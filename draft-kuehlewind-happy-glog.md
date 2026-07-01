---
title: "Happy Eyeballs v3 (HEv3) Event Logging with qlog"
abbrev: "HEv3 qlog"
category: info

docname: draft-kuehlewind-happy-glog-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Heuristics and Algorithms to Prioritize Protocol deploYment"

ipr: trust200902
stand_alone: yes
pi:
  toc: yes
  sortrefs: yes
  symrefs: yes

keyword:
 - qlog
 - HEv3
venue:
  group: "Heuristics and Algorithms to Prioritize Protocol deploYment"
  type: "Working Group"
  mail: "happy@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/happy/"
  github: "mirjak/draft-happy-glog"
  latest: "https://mirjak.github.io/draft-happy-glog/draft-kuehlewind-happy-glog.html"

author:
 -
    fullname: Mirja Kühlewind
    organization: Ericsson
    email: mirja.kuehlewind@ercisson.com

normative:
  QLOG-MAIN:
    I-D.ietf-quic-qlog-main-schema

informative:

...

--- abstract

This document specifies a qlog extension for Happy Eyeballs v3 (HEv3), enabling
logging of dual-stack connection racing behavior. It defines a dedicated
event category, event names, and JSON data structures that capture DNS timing,
candidate discovery, attempt scheduling, fallback timers, racing windows,
success/failure outcomes, and summary metrics.

--- middle

# Introduction

Happy Eyeballs helps application to reduce connection
latency on dual-stack networks. Happy Eyeballs v3 (HEv3) extents racing
From IPv4/IPv6 only to e.g. include TCP+TLS, QUIC/HTTP/3.
Further HEv3 is expected to provide more detailed logging such that connection
failures can be discovered and eventually fixed.
This document defines a qlog extension that provides logging and visibility into HEv3
decision-making and timing.

The extension covers:

* Logging of DNS queries and resolution timing.
* Candidate address discovery and ranking.
* Scheduling, cancellation, or execution of connection attempts.
* Fallback timer management and racing windows.
* Success, failure, and timeout results.
* End-to-end metrics summarizing the HE session.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Event Schema Definition

This document  proposes a qlog schema for HEv3 using the newly defined event schema `urn:ietf:params:qlog:events:hev3`.

## Draft Event Schema Identification

This section is to be removed before publishing as an RFC.

Only implementations of the final, published RFC can use the events belonging to the event schema with the URI `urn:ietf:params:qlog:events:hev3`. Until such an RFC exists, implementations MUST NOT identify themselves using this URI.

Implementations of draft versions of the event schema MUST append the string "-" and the corresponding draft number to the URI. For example, draft 01 of this document is identified using the URI urn:ietf:params:qlog:events:hev3-01.

The namespace identifier itself is not affected by this requirement.


# HE Data Types

## Attempt Target

~~~ cddl
HEAttemptTarget = {
	address: text
	port: uint16
	family: "ipv4" / "ipv6"
	interface: text ?
	path_id: text ?
	alpn: [+ text] ?
	service_name: text ?
	service_priority: uint32 ?
	ech_offered: bool ?

	* $he-attempttarget-extension
}
~~~

The `alpn` field records the negotiated ALPN set for this target derived
from SVCB records. The `service_name` and `service_priority` fields
identify which SVCB ServiceMode record this target originates from.
The `ech_offered` field indicates whether ECH will be attempted on
this connection.

## Policy

~~~ cddl
HEPolicy = {
	resolution_delay_ms: uint32 ?
	preferred_address_family_count: uint32 ?
	connection_attempt_delay_ms: uint32 ?
	min_connection_attempt_delay_ms: uint32 ?
	max_connection_attempt_delay_ms: uint32 ?
	max_parallel_attempts: uint32 ?
	last_resort_local_synthesis_delay_ms: uint32 ?
	preferred_family: "ipv4" / "ipv6" / "auto" ?
	success_definition:
		"tcp_connected" /
		"tls_handshake_complete" /
		"quic_1rtt_ready" ?
	use_historical_rtt: bool ?

	* $he-policy-extension
}
~~~

The fields correspond to the configurable values defined in Section 9 of HEv3:

* `resolution_delay_ms`: Time to wait for preferred-family and SVCB/HTTPS
  records after receiving initial answers (recommended 50ms).
* `preferred_address_family_count`: Number of addresses of the preferred
  family attempted before trying the other family (recommended 1).
* `connection_attempt_delay_ms`: Delay between connection attempts in the
  absence of RTT data (recommended 250ms).
* `min_connection_attempt_delay_ms`: Floor for the connection attempt delay
  (recommended 100ms, MUST NOT be less than 10ms).
* `max_connection_attempt_delay_ms`: Ceiling for the connection attempt delay
  (recommended 2s).
* `max_parallel_attempts`: Maximum number of connection attempts allowed
  in parallel.
* `last_resort_local_synthesis_delay_ms`: Time to wait before querying A
  records for local NAT64 synthesis when AAAA attempts are failing on
  IPv6-only networks (recommended 2s).
* `preferred_family`: The address family assumed to have better connectivity.
* `success_definition`: What constitutes a successful connection establishment.
* `use_historical_rtt`: Whether historical RTT data is used to order
  destinations and adjust attempt delays.

If omitted, `success_definition` defaults to:

* `tcp_connected` for plain TCP stacks
* `tls_handshake_complete` for TCP+TLS stacks 
* `quic_1rtt_ready` for QUIC stacks

## DNS Result

DNS results can represent either address records (A/AAAA) or service records (SVCB/HTTPS).

~~~ cddl
HEDNSResult = HEDNSAddressResult / HEDNSServiceResult

HEDNSAddressResult = {
	type: "address"
	address: text
	family: "ipv4" / "ipv6"
	ttl_s: uint32 ?

	* $he-dnsaddressresult-extension
}

HEDNSServiceResult = {
	type: "service"
	mode: "alias" / "servicemode"
	target_name: text
	priority: uint32 ?
	alpn: [+ text] ?
	no_default_alpn: bool ?
	ech_config: text ?
	ipv4hint: [+ text] ?
	ipv6hint: [+ text] ?
	ttl_s: uint32 ?

	* $he-dnsserviceresult-extension
}
~~~

The `ech_config` field contains a base64-encoded ECHConfigList when present
in the SVCB/HTTPS record. The `mode` field distinguishes AliasMode records
(which require a follow-up query) from ServiceMode records that carry
usable service parameters.

# Event Definitions

Each event uses: 
	`name: "hev3:<event>"`
with the `<event>` type identifier defined below in the section headings.

## Event: config_set

~~~ cddl
HEConfigSet = {
	he_session_id: string
	policy: HEPolicy,
	reason:
		"startup" |
		"network_change" |
		"app_config" |
		"persisted_state"

	* $$he-configset-extension
}
~~~

## Event: config_updated

~~~ cddl
HEConfigUpdated = {
	he_session_id: string
	changed: HEPolicy
	reason:
		"network_change" |
		"admin" |
		"app_hint" |
		"learned_preference"

	* $$he-configupdated-extension
}
~~~

## Event: dns_query_started

~~~ cddl
HEDNSQueryStarted = {
	he_session_id: string
	dns_id: string
	hostname: string
	qtypes: [+ "A" / "AAAA" / "SVCB" / "HTTPS"]
	bootstrap_hint: string ?

	* $$he-dnsquerystarted-extension
}
~~~

## Event: dns_query_finished

~~~ cddl
HEDNSQueryFinished = {
	he_session_id: string
	dns_id: string
	hostname: string
	results: [* HEDNSResult]
	answer_type: "positive" / "negative" / "error" ?
	error_code: text ?
	error_message: text ?
	duration_ms: uint32
	dnssec_validated: bool ?
	is_alias_follow: bool ?

	* $$he-dnsqueryfinished-extension
}
~~~

The `answer_type` field distinguishes positive (non-empty) from negative
(empty, no error) and error responses, which is significant for HEv3's
async resolution logic (Section 4.2). The `dnssec_validated` field indicates
whether the response was cryptographically validated, relevant for
determining whether SVCB-dependent handshakes must be pended (Section 6.3).
The `is_alias_follow` field is set to true when this query was triggered
by an AliasMode SVCB/HTTPS record requiring a follow-up resolution.

## Event: candidate_discovered

~~~ cddl
HECandidateDiscovered = {
	he_session_id: string
	source: "dns" / "cache" / "alt_svc" / "synth" / "preconnect" /
	        "svcb_hint" / "nat64_synthesis"
	target: HEAttemptTarget
	rank: uint32
	group_id: text ?
	supersedes: text ?

	* $$he-candidatediscovered-extension
}
~~~

The `source` value `"svcb_hint"` indicates the address came from ipv4hint or
ipv6hint parameters in a SVCB/HTTPS record. The `"nat64_synthesis"` value
indicates a locally synthesized IPv6 address for NAT64 traversal.

The `group_id` field identifies which protocol/priority group this candidate
belongs to (per Section 5.1 and 5.2 of HEv3). The `supersedes` field
contains the address of an SVCB hint that this candidate replaces once
authoritative A/AAAA records arrive (per Section 7 of HEv3).

## Event: attempt_scheduled

~~~ cddl
HEAttemptScheduled = {
	he_session_id: string
	attempt_id: string
	target: HEAttemptTarget
	scheduled_after_ms: uint32
	priority: uint32
	reason:
		"policy_timer" |
		"dns_completed" |
		"racing_window" |
		"retry"

	* $$he-attemptscheduled-extension
}
~~~

## Event: attempt_started

~~~ cddl
HEAttemptStarted = {
	he_session_id: string
	attempt_id: string
	target: HEAttemptTarget
	transport: "tcp" | "quic"
	ref_event_id: string ?

	* $$he-attemptstarted-extension
}
~~~

QUIC implementations should set `ref_event_id` to the relevant `connectivity:connection_started`.

## Event: attempt_pended

Logged when a connection attempt's TLS handshake must be paused until
SVCB/HTTPS responses are received, as required by Section 6.3 of HEv3
(e.g., when DNS is cryptographically protected and ECH configuration
is expected from SVCB records).

~~~ cddl
HEAttemptPended = {
	he_session_id: string
	attempt_id: string
	reason: "awaiting_svcb" / "awaiting_ech_config" / "dnssec_validation"
	waiting_for: text ?

	* $$he-attemptpended-extension
}
~~~

The `waiting_for` field MAY reference the `dns_id` of the outstanding
SVCB/HTTPS query.

## Event: attempt_resumed

Logged when a previously pended attempt resumes its handshake.

~~~ cddl
HEAttemptResumed = {
	he_session_id: string
	attempt_id: string
	trigger: "svcb_received" / "timeout" / "policy_override"

	* $$he-attemptresumed-extension
}
~~~

## Event: attempt_outcome

~~~ cddl
HEAttemptOutcome = {
	he_session_id: string
	attempt_id: string
	result: "success" | "failure" | "timeout" | "canceled"
	error_code: string ?
	connect_duration_ms: uint32 ?

	* $$he-attemptoutcome-extension
}
~~~

## Event: racing_window_opened

~~~ cddl
HERacingWindowOpened = {
	he_session_id: string
	window_id: string
	max_parallel_attempts: uint32

	* $$he-racingwindowopened-extension
}
~~~

## Event: racing_window_closed

~~~ cddl
HERacingWindowClosed = {
	he_session_id: string
	window_id: string

	* $$he-racingwindowclosed-extension
}
~~~

## Event: fallback_timer_set

~~~ cddl
HEFallbackTimerSet = {
	he_session_id: string
	timer_id: string
	delay_ms: uint32
	for_family: "ipv4" | "ipv6"

	* $$he-fallbacktimerset-extension
}
~~~

## Event: fallback_timer_fired

~~~ cddl
HEFallbackTimerFired = {
	he_session_id: string
  	timer_id: string

	* $$he-fallbacktimerfired-extension
}
~~~

## Event: fallback_timer_canceled

~~~ cddl
HEFallbackTimerCanceled = {
	he_session_id: string,
 	timer_id: string,
	reason: "success” | “abort"

	* $$he-fallbacktimercanceled-extension
}
~~~

## Event: connection_selected

~~~ cddl
HEConnectionSelected = {
	he_session_id: string
	attempt_id: string
	ref_event_id: string ?

	* $$he-connectionselected-extension
}
~~~

## Event: connection_aborted

~~~ cddl
HEConnectionAborted = {
	he_session_id: string
	reason: string

	* $$he-connectionaborted-extension
}
~~~

## Event: metrics

~~~ cddl
HEMetrics = {
	he_session_id: string
	tt_first_success_ms: uint32 ?
	first_success_family: "ipv4" | "ipv6" ?
	attempts_total: uint32
	attempts_success: uint32
	attempts_failure: uint32

	* $$he-metrics-extension
}
~~~

## Conformance Requirements

* Every `attempt_started` MUST have exactly one `attempt_outcome`.
* `connection_selected` MUST reference an attempt whose `result` is `"success"`.
* Exactly one `metrics` event SHOULD appear per HE session.


# Correlation with transport protocols

TCP/TLS implementation can correlate HE events with:

* `tls:handshake_started` 
* `tls:handshake_complete` 
* TCP state transitions (if available in logs)

TBD

# Security Considerations

TBD


# IANA Considerations

This document registers a new entry in the "qlog event schema URIs" registry (created in {Section 15 of QLOG-MAIN}):

Event schema URI:
: urn:ietf:params:qlog:events:hev3

Namespace:
: nev3

Event Types:
:  config_set, config_updated, dns_query_started, dns_query_finished, candidate_discovered, attempt_scheduled, attempt_started, attempt_pended, attempt_resumed, attempt_outcome, racing_window_opened, racing_window_closed, fallback_timer_set, fallback_timer_fired, fallback_timer_canceled, connection_selected, connection_aborted, metrics

Description:
: Event definitions for logging HEv3 events

Reference:
: This document


--- back

# Acknowledgments
{:numbered="false"}

