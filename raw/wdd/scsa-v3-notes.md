# SCSAv3 documentation research

## Purpose and audience

This document is the research companion to the provisional SCSAv3 chapter in
`raw/wdd/scsa-v3.xml`.  It is intended for illumos driver developers and
documentation authors who already understand the basic SCSA target-driver and
HBA-driver model described by *Writing Device Drivers* (WDD).

The WDD chapter owns the reader-facing explanation of SCSAv3.  This file
records the topics investigated, source provenance, representative
implementations, source and manual-page discrepancies, remaining
documentation gaps, and open questions.

Update to scsi.xml and scsihba.xml are a work-in-progress.

## Terms

- [SCSA](scsi.xml#scsi-3)
- [SCSAv3](scsa-v3.xml#scsa-v3)
- [initiator-port (iport)](scsa-v3.xml#scsa-v3-topology)
- [iport map](scsa-v3.xml#scsa-v3-dynamic-iports)
- [target map](scsa-v3.xml#scsa-v3-discovery)
- [SPI addressing](scsa-v3.xml#scsa-v3-overview)
- [complex addressing](scsa-v3.xml#scsa-v3-addressing)

## Provenance policy

Material suitable as a basis for future illumos documentation:

1. CDDL-licensed source, headers, and manual pages in the illumos gate;
2. the existing published WDD sources in `~/ws/il-docbooks`;
3. published OpenSolaris PSARC case material in the illumos ARChive; and
4. original explanations derived from observed illumos interfaces and
   behavior.

For questions about current illumos behavior, the illumos source is the final
authority.  Missing or incomplete manual-page language is a documentation gap;
it does not make behavior implemented by the source unknown.

Later Oracle Solaris documentation can corroborate terminology and provide
links for readers, but must not be copied or closely paraphrased into illumos
documentation.  SCSI specifications can be cited for protocol facts without
reproducing their text.

## Research topics

The investigation covers:

- SCSAv3 terminology and its relationship to traditional SCSA;
- top-level HBA and initiator-port topology;
- static iport registration and dynamic phy and iport maps;
- target discovery, target maps, stabilization, and LUN scanning;
- complex addressing and per-`scsi_device` HBA state;
- self-identifying HBA drivers;
- MPxIO device-versus-path properties;
- framework-managed packet allocation and completion;
- map and reset-notification callback lifetime and concurrency;
- representative native and transitional drivers; and
- migration guidance for new drivers and existing WDD material.

The provisional WDD chapter presents the established programming model.
Evidence, discrepancies, historical context, and unresolved questions for
these topics remain below.

## Representative drivers

### `lmrc`: fixed logical and physical iports

`lmrc` is a compact example of the static-iport model:

1. The top-level attach allocates and fills one transport structure.
2. It calls `scsi_hba_attach_setup()` with `SCSI_HBA_HBA` and
   `SCSI_HBA_ADDR_COMPLEX`.
3. It registers separate `raid` and `phys` iports.
4. Its common `attach(9E)` dispatches according to
   `scsi_hba_iport_unit_address()`.
5. Each iport creates a full-set target map.
6. Firmware discovery reports the complete set of logical or physical
   devices through `set_begin()`, `set_add()`, and `set_end()`.
7. Each iport destroys its target map during detach.

The driver also shows the intended per-device state pattern:
`tran_tgt_init()` associates driver target state with `scsi_device`, and
`tran_tgt_free()` clears it.

Primary files:

- `usr/src/uts/common/io/scsi/adapters/lmrc/lmrc_ddi.c`
- `usr/src/uts/common/io/scsi/adapters/lmrc/lmrc_scsa.c`
- `usr/src/uts/common/io/scsi/adapters/lmrc/lmrc_raid.c`
- `usr/src/uts/common/io/scsi/adapters/lmrc/lmrc_phys.c`

### `pmcs`: dynamic SAS topology

`pmcs` demonstrates the dynamic model:

1. The top-level HBA attaches with `SCSI_HBA_HBA`,
   `SCSI_HBA_ADDR_COMPLEX`, and pHCI-related flags.
2. It creates both an iport map and a SAS phy map.
3. Phy observations group lanes into SAS ports.
4. Phy-map activation adds the corresponding iport to the iport map.
5. Attaching that iport creates its target map.
6. Discovery reports a full set of SCSI and SMP targets below each iport.
7. Removal unwinds target maps, iports, and phy associations.

The driver's synchronization and failure handling are substantially more
complex than the `lmrc` example.  It is useful for understanding dynamic
topology but is not the best introductory worked example.

Primary files:

- `usr/src/uts/common/io/scsi/adapters/pmcs/pmcs_attach.c`
- `usr/src/uts/common/io/scsi/adapters/pmcs/pmcs_subr.c`
- `usr/src/uts/common/io/scsi/adapters/pmcs/pmcs_intr.c`
- `usr/src/uts/common/io/scsi/adapters/pmcs/pmcs_scsa.c`

### `mpt_sas`: transitional design

`mpt_sas` uses iport nodes but retains `SCSI_HBA_TRAN_CLONE`.  It is useful
evidence that iports and complex addressing are separable mechanisms and that
real drivers can occupy a transitional state.  It should not be presented as
the preferred new-driver pattern.

Practitioner guidance is stronger: experienced illumos developers have
repeatedly criticized new drivers for copying `mpt_sas`, describing it in this
context as an example of "how not to do it."  New-driver documentation should
instead use `pmcs` for the complete dynamic SAS model and `smrt`, `pvscsi`, or
`vioscsi` for simpler native-SCSAv3 examples.

## Published PSARC material

### Core cases

- [PSARC/2008/675: SCSI_HBA_ADDR_COMPLEX SCSA data structure
  linkage](https://illumos.org/opensolaris/ARChive/PSARC/2008/675/)
  explains the legacy addressing limitation, deprecates cloned transports,
  and introduces complex addressing and per-`scsi_device` HBA state.
- [PSARC/2008/337:
  scsi-self-identifying](https://illumos.org/opensolaris/ARChive/PSARC/2008/337/)
  introduces the `scsi-self-identifying` driver class for SCSA nexus drivers
  that dynamically create children.  This case addresses discovery plumbing,
  not the complete map architecture.
- PSARC/2008/672, `thebe` SAS/SATA driver, appears to have delivered major
  SCSAv3 implementation work but its case is unpublished in the archive.
  It cannot serve as a documentation source.

### Closely related cases

- [PSARC/2009/125: `scsi_device` property
  interfaces](https://illumos.org/opensolaris/ARChive/PSARC/2009/125/)
  defines path-versus-device property access and unit-address retrieval.
- [PSARC/2009/033:
  `scsi_hba_pkt_comp(9F)`](https://illumos.org/opensolaris/ARChive/PSARC/2009/033/)
  makes packet completion observable by SCSA and connects it explicitly to
  complex addressing and SCSAv3 LUN hotplug.
- [PSARC/2005/680: new `scsi_hba_tran` entry
  points](https://illumos.org/opensolaris/ARChive/PSARC/2005/680/)
  introduces framework-managed packet allocation.
- [PSARC/2006/240: SCSA DMA
  enhancement](https://illumos.org/opensolaris/ARChive/PSARC/2006/240/)
  extends that work to framework-managed DMA allocation and breakup.

### Earlier cases and terminology

The archive indexes list older cases including:

- PSARC/1993/259, SCSI HBA interface;
- PSARC/1996/113, SCSA SCSI-3 enhancements;
- PSARC/1997/012, SCSI HBA hot-plug transport vectors;
- PSARC/1998/334, SCSA extensions for hotplugging support; and
- PSARC/1999/667, SCSI target enumeration.

Those cases are marked unpublished in the public archive.  Their titles can
help locate historical boundaries, but their unavailable content cannot
support detailed claims.  In particular, `SCSA SCSI-3 enhancements` refers to
the SCSI protocol generation and is not evidence of a named SCSAv3 framework.

## Current illumos manuals

The following licensed manual pages collectively provide the public interface
description needed for original documentation:

- [`Intro(9E)`](https://illumos.org/man/9E/Intro)
- [`iport(9)`](https://illumos.org/man/9/iport)
- [`scsi_hba_attach_setup(9F)`](https://illumos.org/man/9F/scsi_hba_attach_setup)
- [`scsi_hba_iport_register(9F)`](https://illumos.org/man/9F/scsi_hba_iport_register)
- [`scsi_hba_iport_unit_address(9F)`](https://illumos.org/man/9F/scsi_hba_iport_unit_address)
- [`scsi_hba_iportmap_create(9F)`](https://illumos.org/man/9F/scsi_hba_iportmap_create)
- [`scsi_hba_tgtmap_create(9F)`](https://illumos.org/man/9F/scsi_hba_tgtmap_create)
- [`scsi_address(9S)`](https://illumos.org/man/9S/scsi_address)
- [`scsi_device(9S)`](https://illumos.org/man/9S/scsi_device)
- [`scsi_hba_tran(9S)`](https://illumos.org/man/9S/scsi_hba_tran)
- [`scsi_hba_pkt_comp(9F)`](https://illumos.org/man/9F/scsi_hba_pkt_comp)

### Missing manual pages

The public `scsi_device_prop_*()` interfaces declared in
`usr/src/uts/common/sys/scsi/conf/device.h` have no manual page:

- scalar getters: `scsi_device_prop_get_int()` and
  `scsi_device_prop_get_int64()`;
- allocated-value lookup functions:
  `scsi_device_prop_lookup_byte_array()`,
  `scsi_device_prop_lookup_int_array()`,
  `scsi_device_prop_lookup_string()`, and
  `scsi_device_prop_lookup_string_array()`;
- update functions: `scsi_device_prop_update_byte_array()`,
  `scsi_device_prop_update_int()`, `scsi_device_prop_update_int64()`,
  `scsi_device_prop_update_int_array()`,
  `scsi_device_prop_update_string()`, and
  `scsi_device_prop_update_string_array()`; and
- `scsi_device_prop_remove()` and `scsi_device_prop_free()`.

These interfaces select device or MPxIO path properties through the
`SCSI_DEVICE_PROP_DEVICE` and `SCSI_DEVICE_PROP_PATH` flags. Documentation
needs to cover their context, return values, allocation ownership, and the
distinction between device and path properties.

The `scsi_hba_tran(9S)` manual is itself incomplete for SCSAv3.  Its structure
description predates the map fields now present in the implementation header,
and it still describes the cloned-transport private fields without documenting
their deprecation or the complex-address replacement.  The current
`scsi_hba_attach_setup(9F)`, `scsi_address(9S)`, and `iport(9)` manuals are
better sources for those parts of the interface.

The current `scsi_hba_pkt_comp(9F)` manual requires that interface for HBAs
using `tran_setup_pkt(9E)`.  PSARC/2009/033 also required it for
complex-address HBAs so SCSA could detect LUN-level hotplug.  The current
implementation performs that detection when a target map and
`SCSI_HBA_ADDR_COMPLEX` are both in use, but the manual does not repeat the
broader PSARC requirement.

## Current source anchors

These locations provide concise implementation evidence for the synthesis:

- `usr/src/man/man9e/Intro.9e:812-833` identifies iports with the third
  framework version and describes repeated `attach(9E)` for iport nodes.
- `usr/src/uts/common/sys/scsi/impl/transport.h:75-80` deprecates cloned
  transport private fields in favor of complex addressing.
- `usr/src/uts/common/sys/scsi/impl/transport.h:297-305` holds the iport-map
  and target-map handles in the transport.
- `usr/src/uts/common/sys/scsi/impl/transport.h:503-544` defines legacy SPI,
  complex addressing, pHCI, HBA-only, and cloned-transport flags.
- `usr/src/uts/common/io/scsi/impl/scsi_hba.c:1340-1349` describes the exact
  meaning and boundary of `SCSI_HBA_HBA`.
- `usr/src/uts/common/io/scsi/impl/scsi_hba.c:5572-5585` connects target-map
  enumeration, complex addressing, packet completion, and LUN-change sense
  data.
- `usr/src/uts/common/io/scsi/impl/scsi_hba.c:8276-8380` selects legacy SPI,
  SCSAv3 target-map, and SCSAv3 iport-map configuration paths.

## Documentation status and remaining gaps

The illumos WDD is available at
[`https://illumos.org/books/wdd/`](https://illumos.org/books/wdd/).
The local DocBook sources now divide responsibility as follows:

- `raw/wdd/scsi.xml` and `raw/wdd/scsihba.xml` retain the traditional target
  and HBA programming material and contain focused notes pointing to modern
  differences;
- `raw/wdd/scsa-v3.xml` provides the coherent reader-facing description of
  modern topology, discovery, addressing, packet interfaces, and MPxIO
  properties; and
- this research file retains evidence, source discrepancies, implementation
  examples, and unresolved questions.

Remaining documentation work includes:

1. The SCSAv3 chapter remains a provisional delta rather than a complete
   replacement for the traditional HBA programming guide.
2. The traditional HBA example still demonstrates cloned transports rather
   than a native SCSAv3 driver.
3. The target-driver probe example reads `a_target` and `a_lun`; a note warns
   that this is incompatible with complex addressing, but the example has not
   been modernized.
4. Map callback concurrency and map-destruction quiescence are established
   from source but are not yet documented in the WDD or manual pages.
5. Reset-notification cancellation does not quiesce an already selected
   callback, but the manuals do not state that lifetime rule.
6. The historical boundary between mandatory, optional, and adjacent SCSAv3
   interfaces remains uncertain.
7. The material lacks a small pedagogical native-SCSAv3 driver example.

## Additional lifecycle and callback research

### Source availability

The public PSARC indexes identify several relevant cases but do not publish
their case materials:

- [PSARC/1997/012, SCSI HBA Hot-Plug Transport
  Vectors](https://www.workingcode.com/arc/caselog/PSARC/1997/);
- [PSARC/1998/334, SCSA extensions for hotplugging
  support](https://www.workingcode.com/arc/caselog/PSARC/1998/); and
- [PSARC/1999/667, SCSI target
  enumeration](https://www.workingcode.com/arc/caselog/PSARC/1999/).

PSARC/2008/672 is also absent from the public case archive.  The integrating
illumos commit and the archived Sun bug records provide more useful evidence
than the unpublished case indexes:

- [the SCSAv3 integrating
  commit](https://github.com/illumos/illumos-gate/commit/4c06356b0f0fffb4fc1b6eccc8e5d8e2254a84d6);
  and
- [Sun bug 6726867,
  SCSAv3](https://www.illumos.org/opensolaris/bugdb/6726867.json).

The bug record describes the intended framework boundary.  An HBA reports
hardware observations such as changes in accessible target ports, while SCSA
handles their Solaris-specific configuration consequences.  It also lists
dynamic iports, device-versus-path properties, string-based state, and
framework handling of common SCSI operations among the project goals.  It does
not provide a normative list of required SCSAv3 interfaces.

Later Oracle WDD material was also checked.  It includes some complex-address
structure fields but otherwise retains the traditional HBA presentation and
does not specify the callback-lifetime or map-destruction contracts below.

### Map configuration, callbacks, and teardown

The current target-map implementation configures and unconfigures different
target addresses concurrently by default.  `scsi_hba_tgtmap_create()` selects
`DAMAP_MTCONFIG` unless the `scsi-enumeration` property disables
multithreaded target enumeration.  `damap` then uses separate task-queue work
for each address and waits for the temporary task queue as a group.  Therefore
driver activation and deactivation callbacks for different target addresses
can overlap in the current implementation.  Iport-map and phy-map
configuration instead use `DAMAP_SERIALCONFIG`.

The implementation and archived bug records establish these additional
behaviors:

- [Sun bug 6944220](https://www.illumos.org/opensolaris/bugdb/6944220.json)
  added a synchronization window between initial map creation, the HBA's first
  observations, and early `bus_config` requests.  Outside that initial window,
  target-map configuration is normally driven by hotplug reports.
- [Sun bug 6880942](https://www.illumos.org/opensolaris/bugdb/6880942.json)
  corrected failed enumeration so that an address does not remain active
  without a later re-enumeration attempt.
- [Sun bug 6915456](https://www.illumos.org/opensolaris/bugdb/6915456.json)
  states that failed device configuration causes SCSA to invoke the target-map
  deactivation callback.  The HBA can use that callback to decide whether to
  report the target again for another attempt.
- [Sun bug 6899010](https://www.illumos.org/opensolaris/bugdb/6899010.json)
  extended deactivation callbacks to reported addresses that are canceled or
  replaced before they stabilize.
- [Sun bug 6848196](https://www.illumos.org/opensolaris/bugdb/6848196.json)
  led to multithreaded unconfiguration of independent addresses.

`damap_destroy()` marks the map as being destroyed, rejects new reports, waits
for pending stabilization, cancels its timer, and synchronously unconfigures
and deactivates the remaining stable addresses before freeing the map.  Its
multithreaded activation and deactivation helpers wait for their task queues
before returning.  The source therefore establishes that destruction quiesces
work owned by the map before returning.  Callers must separately ensure that
no thread uses the map handle after destruction frees it.

The public map manual pages document that target deactivation follows
target-driver detach and that iport-map destruction detaches associated
iports.  They do not say that callbacks for separate target addresses can
overlap, state what locks a callback may acquire, define callback reentrancy,
or explicitly promise that map destruction is a callback-quiescence barrier.

### Reset-notification cancellation

The current `scsi_reset_notify(9F)` and `tran_reset_notify(9E)` manuals say
that `SCSI_RESET_CANCEL` cancels a notification request.  They do not say that
it waits for a callback already selected for invocation.

The implementation in
`usr/src/uts/common/io/scsi/impl/scsi_reset_notify.c` copies the registered
callback and argument while holding the HBA mutex, releases that mutex, and
then invokes the copy.  Cancellation removes and frees the registration under
the same mutex, but it cannot retract or wait for a callback already copied.
Consequently, successful cancellation can overlap a callback that has not yet
returned and can even be followed by invocation of a previously copied
callback.

The general WDD detach guidance requires a driver to cancel or wait for
callbacks before releasing instance resources.  The source establishes that
`SCSI_RESET_CANCEL` alone does not provide that wait.  A target driver that
can overlap reset notification with teardown must provide its own
synchronization, or the framework must be changed to provide a quiescing
cancellation operation.  The missing manual-page language is a documentation
defect, not an unresolved question about current behavior.

## Open research questions

### SCSAv3 scope and interface model

1. Is there a published design document for the iport, phy-map, iport-map, and
   target-map framework outside PSARC/2008/672?
2. Which exact interface set did the original SCSAv3 project consider
   mandatory, optional, or merely adjacent?
3. Should a modern HBA guide require both `SCSI_HBA_HBA` and
   `SCSI_HBA_ADDR_COMPLEX`, or describe supported transitional combinations?

### Documentation scope and examples

4. Which interfaces have current examples suitable for a small pedagogical
   driver rather than a production-driver excerpt?
5. How should MPxIO be introduced without turning an SCSAv3 guide into a full
   multipathing design document?
6. Which categorical statements in the target-driver chapter remain valid
   from the target driver's perspective even when the HBA discovers and
   creates the device node dynamically?
7. Where should the source-defined map concurrency, map-destruction
   quiescence, and reset-notification lifetime rules be documented?  The map
   rules belong with the SCSAv3 interfaces, while reset notification is shared
   traditional SCSA behavior and probably belongs in the existing SCSI target
   and HBA material and their manual pages.

## Possible later work

- Decide whether the provisional SCSAv3 chapter remains separate or is merged
  into the traditional SCSI HBA material.
- Develop a small native-SCSAv3 example or carefully selected driver excerpts.
- Document map callback concurrency and destruction guarantees.
- Correct the reset-notification lifetime contract in the appropriate WDD and
  manual-page material.
- Prepare abbreviated SCSA target-driver and HBA-driver README files for
  placement near the illumos-gate source.
