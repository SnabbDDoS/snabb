### PCI (lib.hardware.pci)

The `lib.hardware.pci` module provides functions that abstract common
operations on PCI devices on Linux. In order to drive a PCI device using
[Direct memory access (DMA)](https://en.wikipedia.org/wiki/Direct_memory_access)
one must:

1. Unbind the PCI device using `pci.unbind_device_from_linux`.
2. Enable PCI bus mastering for device using `pci.set_bus_master` in
   order to enable DMA.
3. Memory map PCI device configuration space using `pci.map_pci_memory`.
4. Control the PCI device by manipulating the memory referenced by the
   pointer returned by `pci.map_pci_memory`.
5. Disable PCI bus master for device using `pci.set_bus_master`.
6. Unmap PCI device configuration space using `pci.close_pci_resource`.

The correct ordering of these steps is absolutely critical.


— Variable **pci.devices**

An array of supported hardware devices. Must be populated by calling
`pci.scan_devices`. Each entry is a table as returned by
`pci.device_info`.

— Function **pci.canonical** *pciaddress*

Returns the canonical representation of a PCI address. The canonical
representation is preferred internally in Snabb and for
presenting to users. It shortens addresses with leading zeros like
this: `0000:01:00.0` becomes `01:00.0`.

— Function **pci.qualified** *pciaddress*

Returns the fully qualified representation of a PCI address. Fully
qualified addresses have the form `0000:01:00.0` and so this function
undoes any abbreviation in the canonical representation.

— Function **pci.scan_devices**

Scans for available PCI devices and populates the `pci.devices` table.

— Function **pci.device_info** *pciaddress*

Returns a table containing information about the PCI device by
*pciaddress*. The table has the following keys:

* `pciaddress`—String denoting the PCI address of the
  device. E.g. `"0000:83:00.1"`.
* `vendor`—Identification string e.g. `"0x8086"` for Intel.
* `device`—Identification string e.g. `"0x10fb"` for 82599 chip.
* `interface`—Name of Linux interface using this device e.g. `"eth0"`.
* `status`—String denoting the Linux operational status, or `nil` if not
  known.
* `driver`—String denoting the Lua module that supports this hardware
  e.g. `"apps.intel.intel10g"`.
* `usable`—String denoting if the device was suitable to use when
  scanned. One of `"yes"` or `"no"`.

— Function **pci.which_driver** *vendor*, *model*

Returns the module name for a suitable device driver (if available) for a
device of *model* from *vendor*.

— Function **pci.unbind_device_from_linux** *pciaddress*

Forces Linux to unbind the device identified by *pciaddress* from any
kernel drivers.

— Function **pci.set_bus_master** *pciaddress*, *enable*

Enables or disables PCI bus mastering for device identified by
*pciaddress* depending on whether *enable* is a true or a false
value. PCI bus mastering must be enabled in order to perform DMA on the
PCI device.

— Function **pci.map_pci_memory** *pciaddress*, *n*

Memory maps configuration space *n* of PCI device identified by
*pciaddress*. Returns a pointer to the memory mapped region and a file
descriptor of the opened sysfs resource file. PCI bus mastering must be
enabled on device identified by *pciaddress* before calling his function.

— Function **pci.close_pci_resource** *file_descriptor*, *pointer*

Closes memory mapped *file_descriptor* of sysfs resource file and unmaps
it from *pointer* as returned by `pci.map_pci_memory`.


### Register (lib.hardware.register)

The `lib.hardware.register` module provides an abstraction for hardware
device registers. This abstraction can be used to declaratively specify
and conveniently manipulate structured memory regions via DMA. The
functions `register.define` and `register.define_array` construct
`Register` objects based on a *register description* string. The
resulting `Register` objects can be used to manipulate the defined
registers using the methods `Register:read`, `Register:write`,
`Register:set`, `Register:clr`, `Register:wait` and `Register:reset`
(exact set depends on the *register mode*).

A register description is a string with one `Register` object definition
per line. A `Register` object definition must be expressed using the
following grammar:

```
Register   ::= Name Offset Indexing Mode Longname
Name       ::= <identifier>
Indexing   ::= "-"
           ::= "+" OffsetStep "*" Min ".." Max
Mode       ::= "RO" | "RW" | "RC"
Longname   ::= <string>
Offset ::= OffsetStep ::= Min ::= Max ::= <number>
```

A `Register` object definition is made up of the following properties:

* *Name*—A string to be used to refer to the `Register` object. Must
  be a valid Lua identifier, e.g. `"foo"`, `"foo_bar"`, `"FOO"` etc.
* *Offset*—Integer specifying the offset from the base pointer (as
  supplied to `register.define` and `register.define_array`).
* *Indexing*—Optional. Three integers specifying the offset step as well
  as minimum and maximum indexes in bytes.
* *Mode*—One of `"RO"`, `"RW"` or `"RC"` standing for *read-only*,
  *read-write* and *counter* modes respectively. Counter mode is for
  counter registers that clear back to zero when read.
* *Longname*—A string describing the register (used for
  self-documentation).

For instance, the following `Register` object definition defines a
register range "TXDCTL" in read-write mode starting at offset 0x06028
with 128 registers each of length 0x40.

```
TXDCTL 0x06028 +0x40*0..127 RW Transmit Descriptor Control
```

The next example defines a singular register "TPT" in counter mode
located at offset 0x01428.

```
TPT 0x01428 - RC Total Packets Transmitted
```

— Function **register.define** *description*, *table*, *base_pointer*,
*n*

Creates `Register` objects for *description* relative to
*base_pointer*. The resulting `Register` objects will become a named
entries in *table* using the names defined in *description*. If an entry
in *description* defines an indexing range then *n* specifies the index
of the register within that range. *N* defaults to 0.

— Function **register.define_array** *description*, *table*,
*base_pointer*

Creates `Register` objects for *description* relative to
*base_pointer*. The resulting `Register` objects will become a named
entries in *table* using the names defined in *description*. If an entry
in *description* defines an indexing range, an array of `Register`
objects will be created instead of a singular `Register` object.

— Function **register.dump** *table*

Prints a pretty-printed register dump of a *table* of registers.

— Method **Register:read**

Returns the value of register. For convenience register objects can be
called without arguments instead of calling
`Register:read`. E.g. `reg:read()` is equivalent to `reg()`.

— Method **Register:write** *value*

Sets the value of register to *value*. Only available on registers in
read-write mode. For convenience register objects can be called with an
argument instead of calling `Register:write`. E.g. `reg:write(value)` is
equivalent to `reg(value)`.

If register is in counter mode it is assumed that the register will be
reset to zero upon reading. The read value is added to a *register
accumulator* and the sum of all reads is returned.

— Method **Register:set** *bitmask*

Sets bits of register according to *bitmask*. Only available on registers
in read-write mode.

— Method **Register:clr** *bitmask*

Clears bits of register according to *bitmask*. Only available on
registers in read-write mode.

— Method **Register:wait**  *bitmask*, *value*

Blocks until applying *bitmask* to the register equals *value*. If
*value* is not supplied blocks until all bits in the mask are set
instead. Only available on registers in read-write and read-only modes.

— Method **Register:reset**

Reset the register accumulator to 0. Only available on registers in
counter mode.

— Method **Register:print**

Prints the register state to standard output.
