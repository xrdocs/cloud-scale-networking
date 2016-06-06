---
published: true
date: '2016-05-31 07:45 -0700'
title: IOS XR Pattern Config Replace
permalink: /tutorials/xr-global-config-replace
author: Jose Liste
excerpt: An overview of IOS XR Global Pattern Config Replace feature
tags:
  - iosxr
  - config replace
  - string replace
---

{% include toc icon="table" title="IOS XR Pattern Config Replace" %}  

>
In IOS XR 5.3.1, a customer favorite feature was implemented - **Pattern Configuration Replace**  
>
This feature allows convenient manipulation of router configuration. Consider the following use cases:
>
*  Ever wanted to conveniently move configuration from one interface to another?
*  Or, rather change all repetitions of a given pattern in your router configuration?
>
If so, keep on reading ...
{: .notice--info}


## Introduction

The feature allows the user to replace configuration based on an interface identifier or a string pattern

Under the global config mode, enter the new **replace** keyboard

```
RP/0/0/CPU0:PE1#configure
RP/0/0/CPU0:PE1(config)#replace ?
  interface  replace configuration for an interface
  pattern    replace a string pattern in configuration
```

## Interface-based Replace operation

In the mode, the user specifies the source and destination interfaces respectively

The command also provides a “**dry-run**” keyboard that allows the user to validate the changes that would be performed by the replace operation without issuing an actual commit

```
RP/0/0/CPU0:PE1(config)#replace interface <ifid_1> with <ifid_2> ?
  dry-run  execute the command without loading the replace config
  <cr>
```

### Sub-Interface considerations

Replacing interface "X" with "Y" will also cause all sub-interfaces hosted under "X" (e.g. "X.abc", "X.def") to also be replaced with "Y"  

Example: `replace interface gigabitEthernet 0/0/0/1 with gigabitEthernet 0/0/0/2` would cause sub-interface gigabitEthernet 0/0/0/1.100 to be replaced to  gigabitEthernet 0/0/0/2.100


### Example 1

In this example, the operator wants to move all configuration located under interface gig 0/0/0/0 to interface gig 0/0/0/2  
In addition, all other references to the former interface (for example: in routing protocols, mpls lpd, rsvp, etc) also need to be replaced with the new interface (gig 0/0/0/2)

Below is the original router configuration

```
RP/0/0/CPU0:iosxrv-1#show run

<snip>
!
interface GigabitEthernet0/0/0/0
 description first
 ipv4 address 10.20.30.40 255.255.255.0
!
interface GigabitEthernet0/0/0/2
 description second
 ipv4 address 10.20.50.60 255.255.255.0
!
router ospf 10
 area 0
  interface GigabitEthernet0/0/0/0
   transmit-delay 5
  !
 !
!
mpls ldp
 interface GigabitEthernet0/0/0/0
  igp sync delay on-session-up 5
 !
 
<snip> 
!
end
```

Below the operator runs the replace command using the interface identifiers.  
In the first attempt, the user decides to give the command a try and specifies the "**dry-run**" keyword in order to validate the results and without loading any config changes

Remember that the goal is to move all configuration and references associated with gig 0/0/0/0 to gig 0/0/0/2

```
RP/0/0/CPU0:iosxrv-1(config)#replace interface gigabitEthernet 0/0/0/0 with gigabitEthernet 0/0/0/2 dry-run
no interface GigabitEthernet0/0/0/0
interface GigabitEthernet0/0/0/2
 description first
 ipv4 address 10.20.30.40 255.255.255.0
router ospf 10
 area 0
  no interface GigabitEthernet0/0/0/0
  interface GigabitEthernet0/0/0/2
   transmit-delay 5
mpls ldp
 no interface GigabitEthernet0/0/0/0
 interface GigabitEthernet0/0/0/2
  igp sync delay on-session-up 5
end
```

By specifying the "dry-run" keyword, we can see below that NO config changes were actually loaded into the candidate config buffer

```
RP/0/0/CPU0:iosxrv-1(config)#
RP/0/0/CPU0:iosxrv-1(config)#show
Thu May 26 05:48:51.519 UTC
Building configuration...
!! IOS XR Configuration 6.1.1.14I
end
```

Now, the operator is ready to proceed with the change. The replace operation is run without using the "dry-run" keyword

```
RP/0/0/CPU0:iosxrv-1(config)#replace interface gigabitEthernet 0/0/0/0 with gigabitEthernet 0/0/0/2
Loading.
365 bytes parsed in 1 sec (357)bytes/sec
RP/0/0/CPU0:iosxrv-1(config)#
```
```
RP/0/0/CPU0:iosxrv-1(config)#show commit changes diff
Thu May 26 07:02:34.026 UTC
Building configuration...
!! IOS XR Configuration 6.1.1.14I
-  interface GigabitEthernet0/0/0/0
-   description first
-   ipv4 address 10.20.30.40 255.255.255.0
   !
   interface GigabitEthernet0/0/0/2
<-  description second
+>  description first
<-  ipv4 address 10.20.50.60 255.255.255.0
+>  ipv4 address 10.20.30.40 255.255.255.0
   !
   router ospf 10
    area 0
-    interface GigabitEthernet0/0/0/0
-     transmit-delay 5
     !
+    interface GigabitEthernet0/0/0/2
+     transmit-delay 5
     !
    !
   !
   mpls ldp
-   interface GigabitEthernet0/0/0/0
-    igp sync delay on-session-up 5
    !
+   interface GigabitEthernet0/0/0/2
+    igp sync delay on-session-up 5
    !
   !
end

RP/0/0/CPU0:iosxrv-1(config)#
RP/0/0/CPU0:iosxrv-1(config)#commit
Thu May 26 07:02:48.775 UTC
RP/0/0/CPU0:iosxrv-1(config)#
RP/0/0/CPU0:iosxrv-1(config)#exit
```
After committing the configuration, we can see the new configuration and updated references to interface gig 0/0/0/2

```
RP/0/0/CPU0:iosxrv-1#show run

<snip>
!
interface GigabitEthernet0/0/0/2
 description first
 ipv4 address 10.20.30.40 255.255.255.0
!
interface GigabitEthernet0/0/0/3
 shutdown
!
router ospf 10
 area 0
  interface GigabitEthernet0/0/0/2
   transmit-delay 5
  !
 !
!
mpls ldp
 interface GigabitEthernet0/0/0/2
  igp sync delay on-session-up 5
 !
!
end
```

### Example 2:
In the previous example, both interfaces had the same configuration statements (description and IPv4 address). With the replace operation, the config from interface gig 0/0/0/1 was moved and effectively merged with the config under interface gig 0/0/0/2

This example will cover the case where the original and new interfaces have "different" configuration statements. This time the user desires to only apply the config from the original interface and not to merge

We start with an initial config where interface gig 0/0/0/0 and gig 0/0/0/2 have different configuration statements (note the presence of a non-default mtu on gig 0/0/0/2)  

```
RP/0/0/CPU0:iosxrv-1#show run

<snip>
!
interface GigabitEthernet0/0/0/0
 description first
 ipv4 address 10.20.30.40 255.255.255.0
!
interface GigabitEthernet0/0/0/2
 description second
 mtu 9000
 ipv4 address 10.20.50.60 255.255.255.0
!
<snip>
 !
!
end
```

To achieve the goal, the user first deletes the interface gig 0/0/0/2 by using the "no" command
And subsequently follow it with the "replace" operation

```
RP/0/0/CPU0:iosxrv-1#conf t
Thu May 26 06:55:50.674 UTC
RP/0/0/CPU0:iosxrv-1(config)#no interface GigabitEthernet0/0/0/2
RP/0/0/CPU0:iosxrv-1(config)#
RP/0/0/CPU0:iosxrv-1(config)#replace interface gigabitEthernet 0/0/0/0 with gigabitEthernet 0/0/0/2
Loading.
365 bytes parsed in 1 sec (357)bytes/sec
RP/0/0/CPU0:iosxrv-1(config)#
```

Observe below the diffs applied on the candidate config buffer. Note that in fact, the target interface gigabitEthernet 0/0/0/2 was not deleted (even though the "no interface" command was issued) but instead the minimal set of changes were automatically applied by the backend; i.e. description and IPv4 addresses were updated and non-default MTU was automatically removed

>
With a new behavior introduced in IOS XR 5.3.2, a **DELETE** followed by a **RECREATE** of an interface translates to a **SET** of minimal changes between original and target interface configuration. This way the user does not have to one-by-one remove unwanted configurations while avoiding unnecessary interface flaps
>
Stay tune for an upcoming tutorial detailing this new behavior !!
{: .notice--warning}

```
RP/0/0/CPU0:iosxrv-1(config)#show commit changes diff
Thu May 26 06:56:44.230 UTC
Building configuration...
!! IOS XR Configuration 6.1.1.14I
-  interface GigabitEthernet0/0/0/0
-   description first
-   ipv4 address 10.20.30.40 255.255.255.0
   !
   interface GigabitEthernet0/0/0/2
<-  description second
+>  description first
-   mtu 9000
<-  ipv4 address 10.20.50.60 255.255.255.0
+>  ipv4 address 10.20.30.40 255.255.255.0
   !
   router ospf 10
    area 0
-    interface GigabitEthernet0/0/0/0
-     transmit-delay 5
     !
+    interface GigabitEthernet0/0/0/2
+     transmit-delay 5
     !
    !
   !
   mpls ldp
-   interface GigabitEthernet0/0/0/0
-    igp sync delay on-session-up 5
    !
+   interface GigabitEthernet0/0/0/2
+    igp sync delay on-session-up 5
    !
   !
end

RP/0/0/CPU0:iosxrv-1(config)#
RP/0/0/CPU0:iosxrv-1(config)#commit
Thu May 26 06:56:54.169 UTC
RP/0/0/CPU0:iosxrv-1(config)#
```

Lastly, below we can see the resulting configuration completely replacing the configuration under interface gigabitEthernet 0/0/0/2 with the one previously found on gigabitEthernet 0/0/0/1

```
RP/0/0/CPU0:iosxrv-1#show run

<snip>
!
interface GigabitEthernet0/0/0/2
 description first
 ipv4 address 10.20.30.40 255.255.255.0
!
!
router ospf 10
 area 0
  interface GigabitEthernet0/0/0/2
   transmit-delay 5
  !
 !
!
mpls ldp
 interface GigabitEthernet0/0/0/2
  igp sync delay on-session-up 5
 !
!
end

RP/0/0/CPU0:iosxrv-1#
```

## Pattern-based Replace operation

In the second mode of operation, the user can provide string patterns with the "replace" command

As it was the case before, the command provides a “**dry-run**” keyboard that allows the user to validate the changes that would be performed by the replace operation without issuing an actual commit

```
RP/0/0/CPU0:PE1(config)#replace pattern ?
  regex-string  pattern to be replaced within single quotes
  
RP/0/0/CPU0:PE1(config)#replace pattern 'string_1' with 'string_2' ?
  dry-run  execute the command without loading the replace config
  <cr>  
```  

### REGEX string matching considerations

The input entered in the replace command is considered a **regex string**

So for example, if you are trying to replace an IPv4 address (e.g. 1.2.3.4), remember to escape the '.' as otherwise it would match any character

Using an IMPROPER input regex string would match undesired statements  
For example, consider a router configuration that includes an interface with ipv4 address 1.2.3.4 and an interface description 10203040  

The following string pattern will match occurrences of 1.2.3.4 as well as 10203040

```
replace pattern '1.2.3.4' with '25.26.27.28' ---> *** DO NOT USE THIS ***
```

The following string pattern will match ONLY occurrences of 1.2.3.4

```
replace pattern '1\.2\.3\.4' with '25.26.27.28' ---> *** USE THIS INSTEAD ***
```

>
Renaming certain configuration objects with replace operation MAY fail to commit due to interdependency checks performed by IOS XR
>
For example, renaming a class-map that is referenced by a policy-map will not go thru commit due to classmap interdependecy on policy-map; semantic error raised: "% Object is in use: Class-map "CMAP-TEST" of type "qos" is used by policy-map(s). Delete failed."
{: .notice--warning} 


## Example 3:
In this example, we will use string pattern replace to move the configuration under Bundle-Ether1000 to a new Bundle-Ether2000 interface  

Configuring bundle interfaces requires both configuration of the logical interface itself as well as the bundle members. Using the "replace interface" command alone as previous examples, would have achieved the former but not the latter. Therefore, we would turn to "replace pattern" to achieve our goal

Below is the original router configuration  

```
RP/0/0/CPU0:iosxrv-1#show run

<snip>
interface Bundle-Ether1000
 bandwidth 1990656
 ipv4 address 13.5.6.5 255.255.255.0
 load-interval 30
!
interface GigabitEthernet0/0/0/0
 bundle id 1000 mode active
 shutdown
!
interface GigabitEthernet0/0/0/1
 bundle id 1000 mode active
 shutdown
!
<snip>
end
```

Repetitions of pattern '1000' are modified to '2000'  
User specifies the "**dry-run**" keyboard to validate the changes without an actual commit

```
RP/0/0/CPU0:iosxrv-1#conf t
RP/0/0/CPU0:iosxrv-1(config)#replace pattern '1000' with '2000' dry-run
no interface Bundle-Ether1000
interface Bundle-Ether2000
 bandwidth 1990656
 ipv4 address 13.5.6.5 255.255.255.0
 load-interval 30
interface GigabitEthernet0/0/0/0
 no bundle id 1000 mode active
 bundle id 2000 mode active
interface GigabitEthernet0/0/0/1
 no bundle id 1000 mode active
 bundle id 2000 mode active
end
```

Replace command is re-executed without the "dry-run" keyboard

```
RP/0/0/CPU0:iosxrv-1(config)#replace pattern '1000' with '2000'
Loading.
319 bytes parsed in 1 sec (312)bytes/sec
```

Below observe the changes in the target configuration buffer

```
RP/0/0/CPU0:iosxrv-1(config)#show commit changes diff
Fri May 27 08:13:38.485 UTC
Building configuration...
!! IOS XR Configuration 6.1.1.14I
-  interface Bundle-Ether1000
-   bandwidth 1990656
-   ipv4 address 13.5.6.5 255.255.255.0
-   load-interval 30
   !
+  interface Bundle-Ether2000
+   bandwidth 1990656
+   ipv4 address 13.5.6.5 255.255.255.0
+   load-interval 30
   !
   interface GigabitEthernet0/0/0/0
<-  bundle id 1000 mode active
+>  bundle id 2000 mode active
   !
   interface GigabitEthernet0/0/0/1
<-  bundle id 1000 mode active
+>  bundle id 2000 mode active
   !
end

RP/0/0/CPU0:iosxrv-1(config)#
RP/0/0/CPU0:iosxrv-1(config)#commit
```

After "commit", then entire Bundle-Ether1000 config (including bundle members) has been moved to Bundle-Ether2000

```
RP/0/0/CPU0:iosxrv-1#show run

<snip>
interface Bundle-Ether2000
 bandwidth 1990656
 ipv4 address 13.5.6.5 255.255.255.0
 load-interval 30
!
interface GigabitEthernet0/0/0/0
 bundle id 2000 mode active
 shutdown
!
interface GigabitEthernet0/0/0/1
 bundle id 2000 mode active
 shutdown
!
<snip>
end
```

As an alternative, note that instead of a single operation, this example could have also been achieved with the following two (2) replace operations
{: .notice--warning}

```
RP/0/0/CPU0:iosxrv-1(config)#replace interface Bundle-Ether1000 with Bundle-Ether2000
RP/0/0/CPU0:iosxrv-1(config)#replace pattern 'bundle id 1000 mode active' with 'bundle id 2000 mode active'
```

## Example 4:
In this example, the goal is to move the configuration under interfaces GigabitEthernet 0/0/0/0, 0/0/0/1 and 0/0/0/2 to interface TenGigE 0/3/0/0, 0/3/0/1 and 0/3/0/2  

Below is the original router configuration

```
RP/0/0/CPU0:iosxrv-1#show run

<snip>
!
interface GigabitEthernet0/0/0/0
 bundle id 2000 mode active
 shutdown
!
interface GigabitEthernet0/0/0/1
 bundle id 2000 mode active
 shutdown
!
interface GigabitEthernet0/0/0/2
 description first
 ipv4 address 10.20.30.40 255.255.255.0
!
<snip>
!
router ospf 10
 area 0
  interface GigabitEthernet0/0/0/2
   transmit-delay 5
  !
 !
!
mpls ldp
 interface GigabitEthernet0/0/0/2
  igp sync delay on-session-up 5
 !
!
end
```

Below we apply the replace pattern command using regex strings `GigabitEthernet0/0/0/([0-2])` and `TenGigE0/3/0/\1`

```
RP/0/0/CPU0:iosxrv-1#conf t
RP/0/0/CPU0:iosxrv-1(config)#replace pattern 'GigabitEthernet0/0/0/([0-2])' with 'TenGigE0/3/0/\1' dry-run
no interface GigabitEthernet0/0/0/0
interface TenGigE0/3/0/0
 bundle id 2000 mode active
 shutdown
no interface GigabitEthernet0/0/0/1
interface TenGigE0/3/0/1
 bundle id 2000 mode active
 shutdown
no interface GigabitEthernet0/0/0/2
interface TenGigE0/3/0/2
 description first
 ipv4 address 10.20.30.40 255.255.255.0
router ospf 10
 area 0
  no interface GigabitEthernet0/0/0/2
  interface TenGigE0/3/0/2
   transmit-delay 5
mpls ldp
 no interface GigabitEthernet0/0/0/2
 interface TenGigE0/3/0/2
  igp sync delay on-session-up 5
```

And there you have it !!!  
I hope that you find this posting useful, but more importantly that you benefit from this new IOS-XR functionality
