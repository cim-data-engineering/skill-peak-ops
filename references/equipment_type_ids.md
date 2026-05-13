# Equipment type → metadata_type_id reference

Static lookup of PEAK equipment type names to the numeric `metadata_type_ids` used by the alert-search URL. Snapshot of `search_equipment_types` (all pages, no filter) — refresh with that tool if a name fails to resolve.

Use this to build Table 2's `Link` URL: look up `AT.equipment_types[0]` (the row's `equipment_type_primary`) in the `Equipment type` column to get `metadata_type_ids`.

If the name doesn't resolve (new type, freeform string), leave the Table 2 `Link` cell empty for that row rather than emitting a link with no `metadata_type_ids` pin (which would silently widen the filter).

| Equipment type | type_id | type_code |
|---|---|---|
| Active Chilled Beams | 10 | ACB |
| Air Cooled Condensers | 13 | ACC |
| Air Curtain | 84 | ACRTN |
| Air Handling Units | 7 | AHU |
| Air to Air Heat Exchanger | 31 | AIR HX |
| BACER | 80 | CIM |
| Bacer Status | 104 | BAC-STATUS |
| Bacer (System) | 105 | SYS-BACER |
| Bathroom Fixtures | 76 | BFX |
| Boiler | 3 | HWB |
| Buffer Tank | 22 | BT |
| Building | 29 | BLD |
| Calorifier | 28 | CAL |
| Car Park Ventilation | 40 | CPV |
| Car Plug | 108 | CPL |
| Central Hot Water Storage | 26 | CEN |
| Chiller | 1 | CH |
| Cleanroom Fan Tower | 82 | CFT |
| Cold Water Booster Pumps | 60 | CWB |
| Combined Heating & Power | 83 | CHaP |
| Compressed Air | 67 | COMP |
| Compressed Air Meter | 78 | CAM |
| Computer Room Air Conditioning Unit | 72 | CRAC |
| Condenser Water Pumps | 6 | CWP |
| Cooling Tower | 2 | CT |
| Demand Management System | 73 | DMS |
| Domestic Cold Water | 81 | DCW |
| Domestic Hot Water | 18 | DHWT |
| Door | 89 | Door |
| Dry Cooler | 55 | DC |
| Dry Cooler Primary Water Pump | 63 | DCP |
| Econet Heat Recovery Unit | 71 | ECO-HR |
| Elevator | 87 | ELVTR |
| Elevator Group | 114 | ELVTR-GRP |
| Evaporative Cooler | 45 | EC |
| Exhaust Air Fans | 32 | EAF |
| Fan Coil Units (Legacy) | 8 | FCU |
| Fire Integration Panel | 58 | FIP |
| Fuel Tank | 101 | FT |
| Fume Cupboard | 94 | FCBD |
| Gas Detector | 86 | GsD |
| Gas Meters | 49 | GM |
| Gas Meters (System) | 69 | SYS-GM |
| Gas Turbines | 27 | GT |
| Generator | 68 | GEN |
| Heat Exchanger | 4 | HE |
| Heat Pump | 51 | HP |
| Heat Recovery Unit | 50 | HRU |
| Hydraulic Pump | 95 | HYD |
| Incinerator Plant | 85 | INCP |
| Kitchen Fans - See Exhaust or Supply Fan | 38 | KF |
| Light | 46 | LT |
| Liquid Nitrogen | 66 | LN |
| Miscellaneous Pump | 74 | MISC-Pump |
| Multi Split Systems | 16 | MSS |
| Natural Ventilation | 39 | NV |
| Packaged Air Conditioning Units | 20 | PAC |
| Passive Chilled Beams | 11 | PCB |
| People | 47 | People |
| Pneumatic System | 19 | PS |
| Power Meters (System) | 21 | SYS-PM |
| Power Sub-Meters | 34 | PSM |
| Primary Chilled Water Pumps | 5 | PCHWP |
| Primary Hot Water Pumps | 23 | PHWP |
| Process Analyzer | 100 | PRA |
| Rain Water Tank | 61 | RWT |
| Refrigeration | 56 | Ref |
| Secondary Chilled Water Pumps (Empty - use SCHWS) | 35 | SCHWP |
| Secondary Chilled Water System | 17 | SCHW |
| Secondary Hot Water System | 25 | SHW |
| Solar Water Heater | 64 | SWH |
| Split Systems | 15 | SS |
| Steam Boiler | 43 | SB |
| Steam Meter | 77 | SM |
| Steam Trap | 96 | STM-TRAP |
| Supply Air Fan | 42 | SAF |
| Switchboard Panel | 112 | SWBD |
| Tenant Condenser Water | 41 | TCW |
| Thermal Energy Meters | 52 | TEM |
| Thermal Labyrinth | 30 | TL |
| Thermostat | 90 | THST |
| Tunnel | 102 | TUN |
| Unassigned | 62 | UNASSIGN |
| Under-Floor Air Distribution | 14 | UFAD |
| Under-Floor Heating System | 79 | UFHWS |
| Uninterruptible Power Supply | 65 | UPS |
| Unit Heater | 106 | UH |
| Variable Air Volume | 9 | VAV |
| Variable Refrigerant Volume Systems | 59 | VRV |
| Variable Speed Drive | 33 | VSD |
| Vertical Transport | 53 | VT |
| Water Meters | 48 | WM |
| Water Meters (System) | 70 | SYS-WM |
| Water Treatment | 75 | WT |
| Weather (System) | 37 | SYS-WEAT |
| Zones | 12 | ZN |
