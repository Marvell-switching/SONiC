# Policy Based Metering

### Table Of Content
- [Policy Based Metering](#policy-based-metering)
    - [Table Of Content](#table-of-content)
    - [Revision](#revision)
    - [Scope](#scope)
      - [Out of Scope](#out-of-scope)
    - [Definitions/Abbreviations](#definitionsabbreviations)
    - [Overview](#overview)
    - [Requirements](#requirements)
      - [Functional Requirements](#functional-requirements)
    - [Scalability Requirements:](#scalability-requirements)
      - [CLI Requirements](#cli-requirements)
    - [Architecture Design](#architecture-design)
    - [High-Level Design](#high-level-design)
      - [Modules and Sub-Modules](#modules-and-sub-modules)
        - [*Image 1: Configuration Flow Overview*](#image-1-configuration-flow-overview)
    - [SAI API](#sai-api)
      - [Usage example:](#usage-example)
    - [Configuration and Management](#configuration-and-management)
      - [Config DB Enhancements](#config-db-enhancements)
        - [New ACL table type](#new-acl-table-type)
        - [Updated ACL Rules Table schema:](#updated-acl-rules-table-schema)
      - [CLI Config Commands](#cli-config-commands)
      - [CLI Show Commands](#cli-show-commands)
      - [YANG Model Enhancements](#yang-model-enhancements)
    - [Warmboot and Fastboot Design Impact](#warmboot-and-fastboot-design-impact)
    - [Restrictions/Limitations](#restrictionslimitations)
    - [Testing Requirements/Design](#testing-requirementsdesign)
      - [Unit Test Cases](#unit-test-cases)
      - [System Test Cases](#system-test-cases)
      - [CLI Level Tests](#cli-level-tests)
      - [DB validation](#db-validation)
    - [Open/Action Items](#openaction-items)
---

### Revision

| Version | Date       | Author                       | Description   |
| ------- | ---------- | ---------------------------- | ------------- |
| 1.0     | 2024-10-13 | Shay Goldshmit (**Marvell**) | Initial Draft |

---
### Scope

This document describes the design of Policy Base Metering (PBM) feature in SONiC.

#### Out of Scope

- [Policer Counter](https://github.com/sonic-net/SONiC/blob/e3f439dcfe2857540a02e4449fce247d4167b621/doc/policer_counter/PolicerCounter-HLD.md#Architecture-Design) - display and check detailed policer statistics (number of packets marked or dropped based on their color).
- [Everflow](https://github.com/sonic-net/SONiC/blob/master/doc/everflow/SONiC%20Everflow%20Always-on%20HLD.pdf) - creating and managing policers.
- [ACL User Defined Table Type Support](https://github.com/sonic-net/SONiC/blob/master/doc/acl/ACL-Table-Type-HLD.md) - ACL feature enhancements with a way user creates with user defined set actions.
---
### Definitions/Abbreviations

| Term | Definition                         |
| ---- | ---------------------------------- |
| ACL  | Access Control List                |
| NAT  | Network Address Translation        |
| SAI  | Switch Abstraction Interface       |
| CIR  | Committed Information Rate         |
| CBS  | Committed Burst Size               |
| PIR  | Peak Information Rate              |
| PBS  | Peak Burst Size                    |

---
### Overview

Policer in networking are responsible for **metering** (Monitoring the rate of traffic) and **marking** (Flagging traffic that exceeds defined limits) traffic based on predefined criteria.
By applying policers to ACL rules, SONiC can effectively control the flow of network traffic, ensuring fairness, optimizing bandwidth utilization, and preventing network congestion.

Usage examples:
- **Security**: Policy-Based Metering can be used to guard against **network storms or DDoS attacks** by limiting traffic rates.
- **Quality of Service (QoS)**: It helps in prioritizing critical traffic (like VoIP or video streams) while limiting non-critical traffic (for example: backup services traffic).
- **Data Center**: ensure fair bandwidth distribution.

---
### Requirements
#### Functional Requirements
- Backward compatibility for existing ACL features - If policer is not set, the system will function as it did previously.
- Ability to config policers with ACL entries.
- Support all existing Policer types (Policer mode, meter_type).
- Support all existing ACL types (ACL table types, ACL stages).
### Scalability Requirements:
- Support for multiple ACL entries with associated policers.
- Query and validate ASIC capabilities dynamically.
#### CLI Requirements
- Bind policers with ACL rules.
- Unbind policer from ACL rules.
- Show ACLs with policers
---
### Architecture Design

No SONiC architecture changes are required as the existing infrastructure is being used.

---
### High-Level Design

#### Modules and Sub-Modules

- **SWSS**
  - ACL-Orch
    - Add/Remove/Manage ACLs.
    - Set or disable policer action.
    - Query from SAI the ACL actions capability.
    - Allow policer action only when capability is enabled (SAI query).
    - Parse and validate policer info.
  - Policer-Orch
    -  Add/Remove/Manage Policers.
    - Validate that the policer already exists and return it's id
    - Prevent from deleting policer that bound to ACLs

##### *Image 1: Configuration Flow Overview*

![alt text](Config_Flow.jpg)


---
### SAI API

Use these **existing** SAI APIs for packet actions and policer association:

| SAI Attribute                           | Description                                                                 |
| --------------------------------------- | --------------------------------------------------------------------------- |
| SAI_ACL_ACTION_TYPE_PACKET_ACTION       | Action type (Forward, Drop, etc) that can be taken in that ACL entry        |
| SAI_ACL_ACTION_TYPE_SET_POLICER         | Action type (policer) that can be taken in that ACL entry                   |
| SAI_ACL_ENTRY_ATTR_ACTION_PACKET_ACTION | Action (Forward, Drop, etc) to be executed on packets matching the ACL rule |
| SAI_ACL_ENTRY_ATTR_ACTION_SET_POLICER   | Action (policer) to be executed on packets matching the ACL rule            |

#### Usage example:
Define the possible actions for the ACL table:
```C++
...
sai_acl_action_type_t action_list[2] = {SAI_ACL_ACTION_TYPE_PACKET_ACTION, SAI_ACL_ACTION_TYPE_SET_POLICER};
acl_attr.value = action_list;
...
sai_acl_api->create_acl_table(...table_attrs...);
```
Specify that the rule action to be taken is policer:
```C++
sai_attribute_t ace_attribute;
ace_attribute.id = SAI_ACL_ENTRY_ATTR_ACTION_SET_POLICER;
ace_attribute.value.aclaction.parameter.oid = policer_id;
...
sai_acl_api->create_acl_entry(...rule_attrs...);
```
---
### Configuration and Management
#### Config DB Enhancements

##### New ACL table type
- When a new ACL is created, SAI API should get a packet-action-list of supported actions that could be used in the rules belonging to this table.
  The system provides pre-defined table types such as L3, L3V6 and MIRROR that contain a default action list per table type.
- To support the new action SET_POLICER, a new ACL table type will be added - TABLE_TYPE_POLICER that contains the SET_POLICER action.
```c++
static acl_table_action_list_lookup_t defaultAclActionList =
{
  ...
  // POLICER
  TABLE_TYPE_POLICER,
  {
    {
      ACL_STAGE_INGRESS,
      {
          SAI_ACL_ACTION_TYPE_SET_POLICER
      }
    },
    {
      ACL_STAGE_EGRESS,
      {
          SAI_ACL_ACTION_TYPE_SET_POLICER
      }
    }
  }
}
...
```
- Note that users can define **custom ACL table types** and specify the desired combination of actions in the configuration.
This mechanism allows for flexibility without requiring code changes for new scenarios, see [ACL User Defined Table Type Support](https://github.com/sonic-net/SONiC/blob/master/doc/acl/ACL-Table-Type-HLD.md) for more info.

##### Updated ACL Rules Table schema:
- The ACL Rules Table schema will be updated with a new **packet_action** parameter - **"policer"** and appropriate
  attribute **"policer_action"** with the value of one of the existing policer object names.
```
key: ACL_RULE_TABLE:table_name:rule_name              ; key of the rule entry in the table,
                                                      ; seq is the order of the rules
                                                      ; when the packet is filtered by the
                                                      ; ACL "policy_name".
                                                      ; A rule is always associated with a policy.
;field        = value
priority      = 1*3DIGIT                              ; rule priority. Valid values range
                                                      ; could be platform dependent

packet_action = "FORWARD"/"DROP"/"REDIRECT"/          ; action when the fields are matched
                "DO_NOT_NAT"/
                "MIRROR"/                             ; (mirror action only available to mirror acl table type)
              * "POLICER"                             ; (policer action only available to policer acl table type)

redirect_action = 1*255CHAR                           ; refer to the destination for redirected packets
                                                      ; it could be:
                                                      : name of physical port.          Example: "Ethernet10"
                                                      : name of LAG port                Example: "PortChannel5"
                                                      : next-hop ip address (in global) Example: "10.0.0.1"

mirror_action = 1*255VCHAR                            ; refer to the mirror session
                                                      ; (only available to mirror acl table type)

* policer_action = 1*255VCHAR                         ; refer to the policer object name
                                                      ; (only available to policer table type)
...
```

#### CLI Config Commands

- **Policers configuration** - No changes (no CLI commands).

- **ACL configuration:**
Two options to **bind policer with ACL** rules:

1. An optional argument **"policer_name"** will be added to the "config acl" commands.
   All rules that belong to that table (as part of the JSON file) will be bound with that policer object.
    ```bash
    # Add/Remove/Update ACL tables
    config acl add table [OPTIONS] <table_name> <table_type> [--policer_name <policer_name>]
    config acl update full [OPTIONS] [--policer_name <policer_name>] <FILE_NAME>
    config acl update incremental [OPTIONS] [--policer_name <policer_name>] <FILE_NAME>

    # note that these commands wrapps "AclLoader" utility script that uses the external "open_config" lib
    ```

2. Use the "config load" command to load the complete JSON file to CONFIG_DB.
   This method enables flexibility to bind different policers to different rules in the same ACL:
    ```JSON
    acl_with_policer_example.json:
    {
        "POLICER_TABLE|M_POLICER_7": {
            "meter_type": "packets",
            "mode": "tr_tcm",
            "color": "aware",
            "cir": "5000",
            "cbs": "5000",
            "green_packet_action": "forward",
            "red_packet_action": "drop"
        },

        "ACL_TABLE|MY_ACL_1": {
            "policy_desc": "Limit some traffic flows",
          * "type": "POLICER",
            "ports": [
                "Ethernet2",
                "Ethernet4",
                "Ethernet7"
            ],
            "OP": "SET"
        },

        "ACL_RULE|MY_ACL_1|MY_RULE_1": {
            "priority": "70",
          * "packet_action": "POLICER",
          * "policer_action": "M_POLICER_7",
            "IP_PROTOCOL": "TCP",
            "SRC_IP": "10.2.130.0/24",
            "DST_IP": "10.5.170.0/24",
            "L4_SRC_PORT_RANGE": "1024-65535",
            "L4_DST_PORT_RANGE": "80-89",
            "OP": "SET"
        }
    }

    - config load acl_with_policer_example.json
    ```

#### CLI Show Commands
```bash
# Show existing policers --> no change
show policer [OPTIONS] [POLICER_NAME]

# Show existing ACL tables --> no change
show acl table [OPTIONS] [TABLE_NAME]

# Show existing ACL rules --> prints are contained the new proposal field
show acl rule [OPTIONS] [TABLE_NAME] [RULE_ID]

# note that these commands wrapps "AclLoader" utility script
```
Example:
```
config acl update full "MY_ACL_1" --policer_name "M_POLICER_7"

admin@sonic:~$ show acl table
Name           Type      Binding    Description                 Stage    Status
-----------    -------   ---------  -------------------------- -------  -----------------
MY_ACL_1     * POLICER   Ethernet2  Limit some traffic flows    Ingress  Inactive
                         Ethernet4
                         Ethernet7

MY_ACL_2       CUSTOM_3  Ethernet8  Limit AND redirect traffic  Ingress  ACTIVE


admin@sonic:~$ show acl rule
Table         Rule          Priority      Actions                    Match
--------      ------------  ----------    -------------------------  ----------------------------
MY_ACL_1      RULE_4        9993        * POLICER: M_POLICER_7       IP_PROTOCOL: 17



MY_ACL_2      RULE_5        9995        * POLICER: M_POLICER_98      L4_SRC_PORT: 80


MY_ACL_2      RULE_6        9994          REDIRECT: Ethernet8        L4_SRC_PORT: 25
```

#### YANG Model Enhancements

sonic-yang-models/yang-templates/sonic-types.yang.j2
```c++
    typedef packet_action{
        type enumeration {
            enum DROP;
            enum ACCEPT;
            enum FORWARD;
            enum REDIRECT;
            enum MIRROR;
            enum DO_NOT_NAT;
          * enum POLICER;
        }
    }
```

sonic-yang-models/yang-templates/sonic-acl.yang.j2:
```c++
    ...
    import sonic-policer {
        prefix policer;
    }
    ...
    container sonic-acl {
        container ACL_RULE {
          ...
            leaf POLICER_ACTION {
                when "current()/../PACKET_ACTION = 'POLICER'";
                type leafref {
                    path "/policer:sonic-policer/policer:POLICER/policer:POLICER_LIST/policer:name";
                }
                description "Policer action applied when PACKET_ACTION_POLICER is True.";
            }
        }
      }
    ...
```

sonic-yang-models/yang-templates/sonic-policer.yang.j2:
```c++
    ...
    import sonic-acl {
      prefix acl;
    }
    ...
    container sonic-policer {
        container POLICER {
        ...
          /* prevent deletion of policer that referenced by ACL rule.
             Note that new policer won't be referenced by any ACL rules initially */
            must "not(../acl:sonic-acl/acl:ACL_RULE/acl:ACL_RULE_LIST[acl:POLICER_ACTION=current()/name])" {
                error-message "Policer cannot be deleted when referenced by an ACL rule.";
            }
        }
    }
```
---
### Warmboot and Fastboot Design Impact
During warmboot or fastboot, both ACL rules and policers configurations are restored from the CONFIG_DB.

---
### Restrictions/Limitations

- Policers must be supported
- PRE/POST INGRESS stage isn't supported (not supported by the existing ACL creation logic)
- Single Action per Rule - each ACL rule typically enforces one action

---
### Testing Requirements/Design
#### Unit Test Cases
- Test ACL-Orchagent and Policer-Orchagent logic for correct processing.
#### System Test Cases
- Ensure correct packet marking based on policer configurations.
- Test different traffic patterns and rates to ensure consistent marking.
- Warm/Fast reboot tests
  - verify that policer configurations are preserved across reboots
  - verify that ACL configurations are preserved across reboots
#### CLI Level Tests
- Verify command run successfully with valid parameter enable/disable.
- Verify command abort with invalid policer parameter.
- Verify command output.
- Verify binding and unbinding policers with ACL rules.
#### DB validation
- Verify CONFIG DB is correctly updated.
---
### Open/Action Items
- ACL-Loader utility uses the an external python library (openconfig_acl) that rely on pre-defined structured models.
  In order to support the new 'policer_action' per rule, openconfig_acl implementation need to be updated.