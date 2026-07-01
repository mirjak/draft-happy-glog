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
  HEV3:
    I-D.ietf-happy-happyeyeballs-v3

informative:

...

--- abstract

This document specifies a qlog extension for Happy Eyeballs v3 (HEv3), enabling
logging of dual-stack and multi-protocol connection racing behavior. It defines
a dedicated event schema, event names, and data structures that capture DNS
resolution timing, SVCB/HTTPS service discovery, candidate sorting and grouping,
connection attempt scheduling and racing, NAT64 prefix discovery,
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

* Logging of DNS queries, SVCB/HTTPS service discovery, and resolution timing.
* Candidate address discovery, grouping, and sorting.
* Scheduling, cancellation, or execution of connection attempts.
* Connection attempt timer management and racing.
* NAT64 prefix discovery and address synthesis.
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
  (recommended 100ms, must not be less than 10ms).
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

All events include a `he_session_id` field that uniquely identifies the
Happy Eyeballs session. This allows correlation of all events belonging
to a single connection establishment attempt.

## Event: config_set

Logged when the HE policy is initially configured for a session.

~~~ cddl
HEConfigSet = {
	he_session_id: text
	policy: HEPolicy,
	reason:
		"startup" /
		"network_change" /
		"app_config" /
		"persisted_state"

	* $$he-configset-extension
}
~~~

The `policy` field contains the full initial policy configuration. The
`reason` field indicates what triggered the configuration:

* `"startup"`: Default policy applied at application start.
* `"network_change"`: Policy set in response to a network transition.
* `"app_config"`: Policy provided by the application.
* `"persisted_state"`: Policy restored from previously saved state.

## Event: config_updated

Logged when the HE policy changes during a session.

~~~ cddl
HEConfigUpdated = {
	he_session_id: text
	changed: HEPolicy
	reason:
		"network_change" /
		"admin" /
		"app_hint" /
		"learned_preference"

	* $$he-configupdated-extension
}
~~~

The `changed` field contains the updated policy configuration.
The `reason` field indicates what triggered the update:

* `"network_change"`: Policy updated due to a network transition.
* `"admin"`: Policy changed by an administrator or system setting.
* `"app_hint"`: Application provided updated preferences.
* `"learned_preference"`: Policy adjusted based on observed behavior
  (e.g., learned RTT data).

## Event: dns_query_started

Logged when a DNS query is initiated as part of hostname resolution.

~~~ cddl
HEDNSQueryStarted = {
	he_session_id: text
	dns_id: text
	hostname: text
	qtypes: [+ "A" / "AAAA" / "SVCB" / "HTTPS"]
	bootstrap_hint: text ?

	* $$he-dnsquerystarted-extension
}
~~~

The `dns_id` uniquely identifies this query within the session and is used
to correlate with the corresponding `dns_query_finished` event. The
`hostname` is the name being resolved. The `qtypes` array lists the DNS
record types being queried. The `bootstrap_hint` field optionally contains
an address hint used to reach the DNS resolver itself.

## Event: dns_query_finished

~~~ cddl
HEDNSQueryFinished = {
	he_session_id: text
	dns_id: text
	hostname: text
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

## Event: nat64_prefix_discovered

Logged when the client discovers a NAT64 prefix for use on an IPv6-only
network (Section 8 of HEv3).

~~~ cddl
HENat64PrefixDiscovered = {
	he_session_id: text
	prefix: text
	prefix_length: uint32
	source: "pref64_ra" / "rfc7050_discovery" / "dns64_inferred"

	* $$he-nat64prefixdiscovered-extension
}
~~~

The `source` field indicates how the prefix was obtained:

* `"pref64_ra"`: Learned from a Router Advertisement (RFC 8781).
* `"rfc7050_discovery"`: Discovered via the well-known name lookup (RFC 7050).
* `"dns64_inferred"`: Inferred from DNS64 behavior on the network.

## Event: candidate_discovered

~~~ cddl
HECandidateDiscovered = {
	he_session_id: text
	source: "dns" / "cache" / "alt_svc" / "synth" / "preconnect" /
	        "svcb_hint" / "nat64_synthesis"
	target: HEAttemptTarget
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

## Event: candidates_sorted

Logged once sufficient DNS answers have been received (per Section 4.2 of
HEv3) and the implementation has applied the three-level sorting algorithm
(Section 5). This event captures the full grouped and ordered candidate list
before racing begins.

~~~ cddl
HECandidatesSorted = {
	he_session_id: text
	groups: [+ HECandidateGroup]

	* $$he-candidatessorted-extension
}

HECandidateGroup = {
	group_id: text
	alpn: [+ text] ?
	ech_available: bool ?
	service_priority: uint32 ?
	candidates: [+ HEAttemptTarget]
}
~~~

The `groups` array is ordered by priority. Within each group:

* `alpn` captures the application protocol set that defines this group
  (Section 5.1 of HEv3).
* `ech_available` indicates whether ECH configuration is available for
  endpoints in this group (Section 5.1 of HEv3).
* `service_priority` reflects the SVCB SvcPriority value for this group
  (Section 5.2 of HEv3). Groups with equal priority are shuffled
  randomly.
* `candidates` is ordered per Section 5.3 of HEv3: RFC 6724 destination
  address selection, historical RTT preferences, and address family
  interleaving applied.

For simple cases without SVCB records, a single group with all candidates
is sufficient.

## Event: candidate_removed

Logged when a candidate address is removed from the list during connection
setup (per Section 7 of HEv3).

~~~ cddl
HECandidateRemoved = {
	he_session_id: text
	target: HEAttemptTarget
	reason: "ttl_expired" / "svcb_hint_replaced" / "dns_negative" / "alpn_mismatch"
	had_active_attempt: bool ?

	* $$he-candidateremoved-extension
}
~~~

The `reason` field indicates why the candidate was removed:

* `"ttl_expired"`: The DNS TTL for this address expired before an attempt
  was started.
* `"svcb_hint_replaced"`: Authoritative A/AAAA records arrived and this
  hint address was absent from them (Section 7.3 of SVCB).
* `"dns_negative"`: A previously positive record was replaced by a negative
  response (e.g., via DNS push notification).
* `"alpn_mismatch"`: The SVCB ALPN set does not contain protocols the
  client supports (Section 6.2 of HEv3).

The `had_active_attempt` field indicates whether an in-progress attempt
exists for this target; per HEv3 Section 7, such attempts are not
canceled.

## Event: candidates_resorted

Logged when the candidate list is re-sorted due to new addresses arriving
mid-race (per Section 7 of HEv3). Re-sorting ensures that address family
interleaving and priority rules are maintained correctly regardless of when
addresses arrive.

~~~ cddl
HECandidatesResorted = {
	he_session_id: text
	trigger: "new_addresses" / "new_svcb" / "dns_push"
	new_order: [+ HEAttemptTarget]

	* $$he-candidatesresorted-extension
}
~~~

The `trigger` field captures why re-sorting occurred:

* `"new_addresses"`: New A/AAAA records arrived (e.g., delayed IPv4 after
  an IPv6-only start).
* `"new_svcb"`: New SVCB/HTTPS ServiceMode records changed grouping or
  priorities.
* `"dns_push"`: A DNS push notification added addresses.

The `new_order` array contains the full re-sorted candidate list as
`HEAttemptTarget` elements. Position in the array implies the new rank.
Only pending candidates (those not yet attempted or currently in progress)
are included.

## Event: attempt_scheduled

Logged when a connection attempt is scheduled to start after a delay.

~~~ cddl
HEAttemptScheduled = {
	he_session_id: text
	attempt_id: text
	target: HEAttemptTarget
	scheduled_after_ms: uint32
	reason:
		"policy_timer" /
		"dns_completed" /
		"resolution_delay_expired" /
		"racing_window" /
		"retry" /
		"last_resort_synthesis"

	* $$he-attemptscheduled-extension
}
~~~

The `attempt_id` uniquely identifies this attempt within the session. The
`scheduled_after_ms` field indicates the delay before the attempt will
start. The `reason` field indicates what triggered scheduling:

* `"policy_timer"`: The Next Connection Attempt Timer fired.
* `"dns_completed"`: DNS resolution completed, enabling the first attempt.
* `"resolution_delay_expired"`: The Resolution Delay timer expired
  (Section 4.2 of HEv3).
* `"racing_window"`: Scheduled as part of parallel racing.
* `"retry"`: A retry of a previously failed attempt.
* `"last_resort_synthesis"`: The Last Resort Local Synthesis Delay expired,
  triggering a fallback A query and NAT64 synthesis (Section 8.4 of HEv3).

## Event: attempt_started

Logged when a connection attempt begins (i.e., the first packet is sent).

~~~ cddl
HEAttemptStarted = {
	he_session_id: text
	attempt_id: text
	target: HEAttemptTarget
	transport: "tcp" / "quic"
	ref_event_id: text ?

	* $$he-attemptstarted-extension
}
~~~

The `transport` field indicates the transport protocol used for this
attempt. The `ref_event_id` field may reference a related event in another
qlog event schema; QUIC implementations should set it to the relevant
`connectivity:connection_started` event.

## Event: attempt_pended

Logged when a connection attempt's TLS handshake must be paused until
SVCB/HTTPS responses are received, as required by Section 6.3 of HEv3
(e.g., when DNS is cryptographically protected and ECH configuration
is expected from SVCB records).

~~~ cddl
HEAttemptPended = {
	he_session_id: text
	attempt_id: text
	reason: "awaiting_svcb" / "awaiting_ech_config" / "dnssec_validation"
	waiting_for: text ?

	* $$he-attemptpended-extension
}
~~~

The `waiting_for` field may reference the `dns_id` of the outstanding
SVCB/HTTPS query.

## Event: attempt_resumed

Logged when a previously pended attempt resumes its handshake.

~~~ cddl
HEAttemptResumed = {
	he_session_id: text
	attempt_id: text
	trigger: "svcb_received" / "timeout" / "policy_override"

	* $$he-attemptresumed-extension
}
~~~

The `trigger` field indicates what unblocked the attempt:

* `"svcb_received"`: The awaited SVCB/HTTPS response arrived.
* `"timeout"`: A timeout expired while waiting; proceeding without the
  expected information.
* `"policy_override"`: The implementation decided to proceed despite not
  receiving the expected response (e.g., opportunistic ECH).

## Event: attempt_outcome

Logged when a connection attempt reaches a terminal state.

~~~ cddl
HEAttemptOutcome = {
	he_session_id: text
	attempt_id: text
	result: "success" / "failure" / "timeout" / "canceled"
	error_code: text ?
	connect_duration_ms: uint32 ?

	* $$he-attemptoutcome-extension
}
~~~

The `result` field indicates the outcome:

* `"success"`: The connection was established per the `success_definition`
  in the policy.
* `"failure"`: The attempt failed (e.g., connection refused, TLS error).
* `"timeout"`: The attempt timed out without completing.
* `"canceled"`: The attempt was canceled because another attempt succeeded.

The `error_code` field contains a protocol-specific error code on failure.
The `connect_duration_ms` field records the time from attempt start to
outcome.

## Event: next_attempt_timer_set

Logged when the Next Connection Attempt Timer is set or adjusted. This
timer controls when the next connection attempt begins (Section 6 of HEv3).

~~~ cddl
HENextAttemptTimerSet = {
	he_session_id: text
	delay_ms: uint32
	reason: "initial" / "rtt_based" / "handshake_progress" ?

	* $$he-nextattemptimerset-extension
}
~~~

The `reason` field captures why the delay was chosen:

* `"initial"`: Using the configured Connection Attempt Delay.
* `"rtt_based"`: Delay derived from historical RTT data or estimated
  retransmission timeout.
* `"handshake_progress"`: Timer extended after partial handshake completion
  (e.g., TCP connected, waiting for TLS — per Section 6.1 of HEv3).

## Event: next_attempt_timer_fired

Logged when the timer fires and the next connection attempt is triggered.

~~~ cddl
HENextAttemptTimerFired = {
	he_session_id: text

	* $$he-nextattemptimerfired-extension
}
~~~

## Event: next_attempt_timer_canceled

Logged when the timer is canceled before firing.

~~~ cddl
HENextAttemptTimerCanceled = {
	he_session_id: text
	reason: "success" / "abort" / "list_exhausted"

	* $$he-nextattemptimercanceled-extension
}
~~~

The `reason` field indicates why the timer was canceled:

* `"success"`: A connection attempt succeeded.
* `"abort"`: The HE session was aborted.
* `"list_exhausted"`: All candidates have been attempted.

## Event: connection_selected

Logged when a successful connection attempt is chosen as the winning
connection for this HE session.

~~~ cddl
HEConnectionSelected = {
	he_session_id: text
	attempt_id: text
	ref_event_id: text ?

	* $$he-connectionselected-extension
}
~~~

The `attempt_id` identifies which attempt won. The `ref_event_id` field
may reference the corresponding transport-level event (e.g., a QUIC
`connectivity:connection_started` or TLS `handshake_complete`).

## Event: connection_aborted

Logged when the entire HE session fails without establishing any connection.

~~~ cddl
HEConnectionAborted = {
	he_session_id: text
	reason: text

	* $$he-connectionaborted-extension
}
~~~

The `reason` field contains a human-readable or implementation-defined
description of why the session was aborted (e.g., "all attempts failed",
"no addresses resolved", "user canceled").

## Event: metrics

Logged at the end of an HE session to provide summary statistics.

~~~ cddl
HEMetrics = {
	he_session_id: text
	tt_first_success_ms: uint32 ?
	first_success_family: "ipv4" / "ipv6" ?
	attempts_total: uint32
	attempts_success: uint32
	attempts_failure: uint32

	* $$he-metrics-extension
}
~~~

The fields capture end-to-end session statistics:

* `tt_first_success_ms`: Time from session start to the first successful
  connection attempt (i.e., time-to-first-byte readiness).
* `first_success_family`: The address family of the first successful attempt.
* `attempts_total`: Total number of connection attempts initiated.
* `attempts_success`: Number of attempts that completed successfully.
* `attempts_failure`: Number of attempts that failed, timed out, or were
  canceled.

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
: hev3

Event Types:
:  config_set, config_updated, dns_query_started, dns_query_finished, nat64_prefix_discovered, candidate_discovered, candidates_sorted, candidate_removed, candidates_resorted, attempt_scheduled, attempt_started, attempt_pended, attempt_resumed, attempt_outcome, next_attempt_timer_set, next_attempt_timer_fired, next_attempt_timer_canceled, connection_selected, connection_aborted, metrics

Description:
: Event definitions for logging HEv3 events

Reference:
: This document


--- back

# Acknowledgments
{:numbered="false"}

