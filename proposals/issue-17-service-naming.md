# Proposal for issue #17: two-label service names with RFC 6763 subtypes

Addresses [SMPTE/ST-2138-10#17](https://github.com/SMPTE/ST-2138-10/issues/17),
per maintainer request to prepare text against clauses 7.4 and 7.5.
This file is proposal text for committee consideration. It changes no PDF;
if accepted, the editors apply it to the draft source.

Conventions: `shall`/`shall not` normative, matching the draft. Every
replacement below quotes the current draft text it supersedes.

---

## 1. Clause 7.4.1 — replace the naming form and value tables

CURRENT (non-conformant, four labels):

> Each service instance shall have an instance name which shall conform to
> the following:
> `<instance-name>._st2138.<api>.<transport>.<network>.local`

PROPOSED:

> Each service instance shall have an instance name which shall conform to
> the following:
> `<instance-name>._st2138.<network>.local`
>
> Where:
> - `<instance-name>` is a human-readable identifier for the service instance
> - `_st2138` identifies the control protocol as SMPTE ST 2138
> - `<network>` specifies the network layer which shall be one of these
>   values:
>   - `_tcp`
>   - `_udp`, for instances using `quic` (see Table 2, `transport`)
>
> The two-label service type keeps every ST 2138 instance discoverable with
> stock DNS-SD tooling: a browse for `_st2138._tcp` (or `_st2138._udp`)
> enumerates all instances on the link, as RFC 6763 section 4.1.2 requires.
>
> API and transport narrowing, previously encoded as extra name labels, shall
> use RFC 6763 section 7.1 subtypes. An instance offering an API shall
> advertise the corresponding subtype:
>
> | API offered | subtype advertised |
> |---|---|---|
> | gRPC | `_grpc._sub._st2138._tcp` |
> | REST | `_rest._sub._st2138._tcp` |
> | WebSocket | `_ws._sub._st2138._tcp` |
>
> A client discovering only a subset (for example, gRPC endpoints) browses
> the subtype; a client discovering everything browses `_st2138._tcp`. The
> advertised Service Instance Name is unchanged by the use of subtypes. TLS
> posture (`http2s` vs `http2`, `https` vs `http`) and QUIC adoption state
> are expressed with the Table 2 `transport` key, not with name labels.
>
> Naming Requirements (unchanged):
> - The instance name shall be unique on the local network.
> - Implementations may include the hostname or another unique identifier.

NOTE (non-normative, for the editors): the Service Name `st2138` is not
present in the IANA registry that RFC 6763 section 4.1.2 reaches via
RFC 6335. Registering it is a process action accompanying this change, not
a protocol behavior — stated here so it is not lost.

## 2. Clause 7.4.2 — replace the record examples

CURRENT PTR/SRV examples use the four-label form. PROPOSED (same router,
same port semantics, conformant names):

> - PTR Record
>
>   The PTR record allows clients to discover all available
>   `_st2138._tcp.local.` services.
>
>   For example, this record:
>
>   `_st2138._tcp.local. PTR router._st2138._tcp.local.`
>
>   Announces an instance called `router` that provides the ST 2138 service.
>   A client seeking only gRPC instances browses
>   `_grpc._sub._st2138._tcp.local.` instead.
>
> - SRV Record
>
>   The SRV record provides connection details for a specific instance:
>
>   For example:
>
>   `router._st2138._tcp.local. SRV 0 0 <port> <hostname>.local.`

TXT record paragraph: replace
`<instance-name>._st2138.<api>.<transport>.<network>.local. TXT <key>=<value>`
with
`<instance-name>._st2138.<network>.local. TXT <key>=<value>`.

## 3. Table 2 — add `api` and `transport` keys

The subtype labels of section 7.4.1 need a single defined vocabulary, shared
with Table 3, so the name and the record cannot drift apart:

| Key | Required? | Description | Example |
|---|---|---|---|
| api | No | The API the instance offers; one of `grpc`, `rest`, `ws`. Mirrors the advertised subtype. | `api=grpc` |
| transport | No | The transport beneath the API; one of `http2s`, `http2`, `https`, `http`, `quic`. | `transport=http2s` |

## 4. Clause 7.5 — replace the Avahi example

The example advertises one instance over two APIs; with subtypes it is one
instance, one type, two subtype attestations:

```xml
<?xml version="1.0" standalone='no'?>
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">
<service-group>
    <name replace-wildcards="yes">router</name>

    <!-- ST 2138, both APIs on one instance; browsers see _st2138._tcp,
         filtered browsers see _grpc._sub or _rest._sub -->
    <service>
        <type>_st2138._tcp</type>
        <subtype>_grpc._sub._st2138._tcp</subtype>
        <subtype>_rest._sub._st2138._tcp</subtype>
        <port>6254</port>
        <txt-record>authz=off</txt-record>
        <txt-record>interface=https://smpte.org/registry/st2138/service</txt-record>
        <txt-record>if_version=2025.1</txt-record>
        <txt-record>api=grpc</txt-record>
        <txt-record>transport=http2s</txt-record>
        <txt-record>sdk=https://github.com/rossvideo/Catena</txt-record>
        <txt-record>sdk_version=cpp-v0.0.7-1</txt-record>
    </service>

</service-group>
```

(Note: the current draft's two `<service>` blocks imply two ports
(6254/8080) for one instance; the replacement keeps a single service entry.
If dual-port operation is intended, that needs its own instance naming rule,
which is out of scope for this issue.)

## 5. Table 3 — correct the value sets

- `api`: add `ws` (the draft's own 7.4.1 lists `_ws` as an API value, but
  Table 3 admits only `gRPC, REST`).
- `transport`: correct `qui` to `quic`.

No other Table 3 change. `service-name`, `hostname`, `port`, address,
authorization, and metadata rows are untouched.

---

## Cross-checks performed (WO acceptance)

- RFC 6763 §4.1.2: proposed names are label-pairs (`_st2138` + `_tcp`/`_udp`).
- RFC 6763 §7.1: subtype form `_x._sub._st2138._tcp` matches the printer
  worked example; instance names unchanged by subtypes.
- RFC 6335: IANA registration noted as editor action, not mandated.
- Table 3 `api`/`transport` values are now a subset of the Table 2 key
  vocabularies — no drift between name, TXT, and registration payload.
- Instance-name uniqueness rule and hostname guidance carried over verbatim.
- No clause outside 7.4.1, 7.4.2, Table 2, 7.5, and Table 3 is touched.

## Verification appendix (primary sources, 2026-09-20)

- RFC 6763 §4.1.2 pair rule + §7.1 subtype mechanics verified verbatim
  against rfc-editor.org rfc6763.txt (not from memory).
- Avahi `<subtype>` (repeatable child of `<service>`) verified against the
  authoritative DTD (`avahi-daemon/avahi-service.dtd`,
  `<!ELEMENT service (type,subtype*,...)>`), so the 7.5 example's two
  subtype elements are schema-valid.
- Draft text extracted from the repo PDF (34 pp); every `_st2138._x._y`
  occurrence located — all live in 7.4.2/7.5, both covered. Table 3 `qui`
  typo and missing `ws` confirmed against extracted text.
- One correction made during audit: an early draft claimed resolvers
  consult the IANA registry; reworded to the process action it is.
