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
area: WIT
workgroup: happy

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
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

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

This document  proposes a qlog schema for HEv3 using the newly defined event schema 'urn:ietf:params:qlog:events:hev3'.

## Draft Event Schema Identification

This section is to be removed before publishing as an RFC.

Only implementations of the final, published RFC can use the events belonging to the event schema with the URI 'urn:ietf:params:qlog:events:hev3'. Until such an RFC exists, implementations MUST NOT identify themselves using this URI.

Implementations of draft versions of the event schema MUST append the string "-" and the corresponding draft number to the URI. For example, draft 01 of this document is identified using the URI urn:ietf:params:qlog:events:hev3-01.

The namespace identifier itself is not affected by this requirement.


# HE Data Types

## Attempt Target

~~~ cddl
HEAttemptTarget = {
	address: string
	port: uint16
	family: "ipv4" | "ipv6"
	interface: string ?
	path_id: string ?

	* $he-attempttarget-extension
}
~~~

## Policy

~~~ cddl
HEPolicy = {
	base_delay_ms: uint32
	jitter_ms: uint32
	max_parallel_attempts: uint32
	prefer_ipv6: bool
	prefer_last_success_family: bool
	ipv6_precedence_offset: int32
	initial_family: "ipv4" | "ipv6" | "auto"
	success_definition:
		"tcp_connected" |
		"tls_handshake_complete" |
		"quic_1rtt_ready"  ?

	* $he-policy-extension
}
~~~

If omitted, `success_definition` defaults to:

* `tcp_connected` for TCP/TLS stacks 
* `quic_1rtt_ready` for QUIC stacks

## DNS Result

~~~ cddl
HEDNSResult = {
	address: string
	family: "ipv4" | "ipv6"
	ttl_s: uint32 ?
	priority: uint32 ?
}
~~~

# Event Definitions

Each event uses: 
	'name: "hev3:<event>"'
with the '<event>' type identifier defined below in the section headings.

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
	qtypes: [ "A" | "AAAA"]+
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
	results: [HEDNSResult]
	error_code: string ?
	error_message: string ?
	duration_ms: uint32

	* $$he-dnsqueryfinished-extension
}
~~~

## Event: candidate_discovered

~~~ cddl
HECandidateDiscovered = {
	he_session_id: string
	source: "dns" | "cache" | "alt_svc" | "synth" | "preconnect"
	target: HEAttemptTarget
	rank: uint32

	* $$he-candidatediscovered-extension
}
~~~

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
:  config_set, config_updated, dns_query_started, dns_query_finished, candidate_discovered, attempt_scheduled, attempt_started, attempt_outcome, racing_window_opened, racing_window_closed, fallback_timer_set, fallback_timer_fired, fallback_timer_canceled, connection_selected, connection_aborted, metrics

Description:
: Event definitions for logging HEv3 events

Reference:
: This document


--- back

# Acknowledgments
{:numbered="false"}

