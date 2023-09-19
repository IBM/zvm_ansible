# zvm_ansible
A set of ansible modules, roles, and playbooks for managing z/VM via SMAPI and zthin. 
These modules requre a Linux virtual machine running on the target z/VM LPAR. The Linux target virtual machine must be authorized to issue SMAPI commands. The Linux target virtual machine must have Feilong zthin installed since the modules use the smcli command.  If you are also an IBM Cloud Infrastructure Center user, you can simply use the existing ICIC Compute node that runs on each managed z/VM LPAR. 

## design
The modules are currently packaged as an ansible collection at https://github.ibm.com/rjbrenn/zvm_ansible_collection

They are kinda a 1:1 wrapper around a pair of create/delete SMAPI calls or virtual RDR management commands. 

You'll end up with a configuration that looks like this: 
![diagram of ansible call path to zvm smapi](/docs/zvm_ansible.png)

When you run the ansible playbooks the rendered python will get pushed to a Linux machine running in the target z/VM partition. The python runs there in that target Linux machine, and in turn uses smcli commands to talk to the SMAPI subsystem in z/VM. Since we are running our rendered python on something other than the desired resultant Linux Virtual Machine in z/VM we use the `ansible_host` to direct ansible to run our modules on that Linux management system.  

You can create a group of linux virtual machines using the following flow:

- Create a set of new virtual machines using `zvm_user.py`  
- Adjust the settings of the above virtual machines as desired using `zvm_update_user.py`
- Create a set of new empty disks of various sizes for the above virtual machines using `zvm_minidisk.py`
  - or: Clone an existing gold image into the above virtual machines using `zvm_clone_disk.py`  
- Direct the new virtual machines to empty their virtual RDR with `zvm_reader_empty.py`  
- Send new network firstboot scripts to the new virtual machines RDRs with `zvm_fileto_reader.py`  
- Start the new virtual machines up with `zvm_startstop_user.py`

Its also possible to spawn openshift clusters using the above overall approach. 

## requirements
z/VM SMAPI must be configured and working, as well as a z/VM directory manager  

ref: https://www.ibm.com/docs/en/zvm/7.3?topic=configuration-setting-up-configuring-server-environment

each z/VM system needs a Linux virtual machine running with permissions to use SMAPI as well as some greater than usual access. We call these linux machines the "Management Access Point" or MAPs - and each one is named to go with the z/VM system it serves - so z/VM LPAR LTICVMA has Linux MAPVMA, z/VM LPAR LTICVM1 has Linux MAPVM1, and so on. 

Our Linux MAP systems have z/VM class D in addition to G so that they can manage the rdr files for linux systems. The MAP Linux system also needs the ability to force and xautolog Linux virtual machines directly instead of via SMAPI for some use cases. We did not want to give z/VM permission class A to our MAP Linux systems because that would be excessive. Instead, we added the class A actions of the XAUTOLOG and FORCE commands to a new privclass T and gave our Linux class T also. 

Our z/VM SYSTEM CONFIG has the following to set up privclass T

    MODIFY CMD XAUTOLOG             IBMCLASS A PRIVCLAS AT
    MODIFY CMD FORCE                IBMCLASS A PRIVCLAS AT

The combination of classes DGT for our MAP Linux systems gives them the z/VM system privileges required for our modules to do their thing. We also enable the LNKNOPAS option so we can optionally dasdfmt new minidisks as they are created by attaching them to the MAP Linux system and running dasdfmt as part of the module action. 

For reference - our MAP Linux virtual machine directory entries look like this: 
```
USER MAPVM1 AUTOONLY 4G 8G GTD
INCLUDE LITPRO
IPL 201
OPTION LNKNOPAS LANG AMENG APPLMON
NICDEF 0F00 TYPE QDIO LAN SYSTEM 10DOTLAN
NICDEF 0F00 VLAN 1284
MDISK 0201 3390 10017 10016 CS2G02 MR
```


We modify the VSMWORK1 AUTHLIST file on SFS space VMSYS:VSMWORK1 to add our MAP Linux virtual machine to the authorization list. We were lazy in our testing and gave ALL ALL permissions - you may wish to be more choosy in your environment. 

each MAP linux system requires zthin from Feilong installed and available in /opt/zthin
Specifically we need the smcli binary since we use that to talk to SMAPI  
ref: https://cloudlib4zvm.readthedocs.io/en/latest/quickstart.html#installation  
ref: https://github.com/openmainframeproject/feilong  
ref: https://openmainframeproject.org/projects/feilong  

A handy end to end set of documentation for setting this all up is the IBM Cloud Infrastructure "Setting up z/VM" documentation at https://www.ibm.com/docs/en/cic/1.2.0?topic=environment-setting-up-zvm   
Pay particular attention to the following sections therin: 
- Special settings for the compute user IDs
- Configuring z/VM SMAPI
- Setup z/VM directory management tool
- Setting up external security manager 


Once you have it all working you ought to be able to run this command from your new MAP Linux machine:
```
# /opt/zthin/bin/smcli Image_Query_DM -T OP1
```

and get a response similar to this: 
```
IDENTITY OP1      AUTOONLY   32M   32M ABCDEFG
INCLUDE IBMDFLT
BUILD ON LTICVM1 USING SUBCONFIG OP1-1
BUILD ON LTICVM3 USING SUBCONFIG OP1-2
BUILD ON LTICVM6 USING SUBCONFIG OP1-3
BUILD ON LTICVM7 USING SUBCONFIG OP1-4
BUILD ON LTICVMA USING SUBCONFIG OP1-5
BUILD ON LTICVMB USING SUBCONFIG OP1-6
BUILD ON LTICVMC USING SUBCONFIG OP1-7
BUILD ON LTICVMD USING SUBCONFIG OP1-8
AUTOLOG AUTOLOG1 MAINT
ACCOUNT IBM
MACH ESA
IPL 190
```


## limitations

SMAPI does not provide an API to manage the vlan tag of a NICDEF statement.  
SMAPI does not provide an API to manage CRYPTO statements for APVIRT or to Dedicate a domain. 

due to the above we have to either:  
a) do full directory parsing and replacement with jinja templated directory stubs   
b) craft a set of prototype directory entries that provide all possible required VLANs and CRYPTO configs     

we felt like B was kinda gross and we'd eventually end up doing A anyway - so thats what our examples show. The `zvm_user.py` module supports creating virtual machines from a prototype though so feel free to use that if it works for you. 

SMAPI does not provide an API to IPL a virtual machine from a specific device: you can effectively `XAUTOLOG SOMEDUDE` but you cannot `XAUTOLOG SOMEDUDE IPL 00C`. We provide samples of how to use `vmcp` to do the xautolog from our ansible target machine as long as you created that extended class T in SYSTEM CONFIG. 

The overall error reporting for situations such as bad usage of memory limits and other syntax issues is not great. Its not specific to our modules, its a function how how SMAPI builds on top of a Directory Manager which in turn runs VM System commands - depending on where the error comes from it may not bubble all the way back to our python module in a usable fashion. You will know it failed, but the reason why will not be obvious.  For example - If you attempt to create a virtual machine whos default memory size is larger than its largest memory size you get an error like the following: 

    <mapvma.fpet.pokprv.stglabs.ibm.com> (0, b'', b'')
    fatal: [jtu0001.fpet.pokprv.stglabs.ibm.com]: FAILED! => {
        "changed": false,
        "invocation": {
            "module_args": {
                "accounting": null,
                "dirfile": "/tmp/jtu001.direct",
                "erasemode": null,
                "exists": true,
                "name": "jtu001",
                "newpass": null,
                "prototype": null
            }
        },
        "msg": "failing return code from smcli is: 1",
        "reason_code": -9,
        "return_code": 1,
        "return_stderr": [],
        "return_stdout": [
            "Defining jtu001 in the directory... ",
            "Failed",
            "  Return Code: 596",
            "  Reason Code: 6213",
            "  Description: ULGSMC5596E Internal directory manager error - product-specific return code : 6213",
            "  API issued : Image_Create_DM"
        ]
    }

RC 596 RS 6213 means "Dirmaint got some CP error when it tried to do something. Examine the DIRMAINT console messages at this time stamp to figure out what the deal is"  We cannot get the Dirmaint console to do that analysis within the python module since theres no SMAPI API for it, so its on the ansible user to coordinate with the z/VM admins to determine what happened. 


The actual messages from DIRECTXA in the DIRMAINT console looks like this for default memory being larger than max memory:



    14:00:06 DIRMAINT LTICVM1. - 2023/07/19; T=0.01/0.01 14:00:06
    14:00:06 DVHREQ2290I Request is: REQUEST 107659 ASUSER VSMWORK2 ADD JTU001
    14:00:06 DVHREQ2288I REQUEST=107659 RTN=DVHREQ MSG=2288 FMT=01 SUBS= ADD ,
    14:00:06 DVHREQ2288I CONT=JTU001 * VSMWORK2
    14:00:06 DVHBXX6213E REQUEST=107659 RTN=DVHBXX MSG=6213 FMT=01 SUBS= 1 2 ,
    14:00:06 DVHBXX6213E CONT=3 4 5 6 7 8  z/VM USER DIRECTORY CREATION PROGRAM - ,
    14:00:06 DVHBXX6213E CONT=VERSION 7 RELEASE 3.0
    14:00:06 DVHBXX6213E REQUEST=107659 RTN=DVHBXX MSG=6213 FMT=01 SUBS= 1 2 ,
    14:00:06 DVHBXX6213E CONT=3 4 5 6 7 8  HCPDIR750I RESTRICTED PASSWORD FILE NOT,
    14:00:06 DVHBXX6213E CONT= FOUND
    14:00:06 DVHBXX6213E REQUEST=107659 RTN=DVHBXX MSG=6213 FMT=01 SUBS= 1 2 ,
    14:00:06 DVHBXX6213E CONT=3 4 5 6 7 8  HCPDIR1776E DEFAULT STORAGE SIZE EXCEED,
    14:00:06 DVHBXX6213E CONT=S MAXIMUM STORAGE SIZE FOR USER JTU001.
    14:00:06 DVHBXX6213E REQUEST=107659 RTN=DVHBXX MSG=6213 FMT=01 SUBS= 1 2 ,
    14:00:06 DVHBXX6213E CONT=3 4 5 6 7 8  EOJ DIRECTORY NOT UPDATED
    14:00:06 DVHADD3212E REQUEST=107659 RTN=DVHADD MSG=3212 FMT=01 SUBS= 2 EX,
    14:00:06 DVHADD3212E CONT=EC DVHBBXXA U
    14:00:06 DVHREQ2289E REQUEST=107659 RTN=DVHREQ MSG=2289 FMT=02 SUBS= 3212,
    14:00:06 DVHREQ2289E CONT= ADD JTU001 *
    
