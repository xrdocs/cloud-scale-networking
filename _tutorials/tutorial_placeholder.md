---
title: IOS XR Global Config Replace
author: Jose Liste
published: true
permalink: "/tutorials/ios-xr-global-config-replace"
excerpt: An introduction to Global Config Replace feature in IOS XR
date: "2016-05-18 15:12 -0700"
position: hidden
---

>
In IOS XR 5.3.1, a customer favorite feature was implemented - **Global Configuration Replace**  
>
This feature allows convenient manipulation of router configuration
>
*  Ever wanted to easily move configuration from one interface to another?
*  Or, rather wanted to change all repetitions of a given pattern in your router configuration?
>
If so, keep on reading ...
{: .notice--info}

## Introduction

The feature allows the user to replace configuration based on an interface identifier or a string pattern

```
RP/0/0/CPU0:PE1#configure
RP/0/0/CPU0:PE1(config)#replace ?
  interface  replace configuration for an interface
  pattern    replace a string pattern in configuration
```

## Interface-based Replace operation

In the mode, the user provides the FROM interface identifier and the TO interface identifier; ifid_1 and ifid_2 respectively in the output below

```
RP/0/0/CPU0:PE1(config)#replace interface <ifid_1> with <ifid_2> ?
  dry-run  execute the command without loading the replace config
  <cr>
```

Note that replacing interface "X" with "Y" will also cause sub-interfaces hosted under "X" (e.g. interface "X.abc", "X.def") to also be replaced with "Y"
Example: replace Gig0/0/0/1 with Gig0/0/0/2 would cause sub-interface Gig0/0/0/1.100 to be replaced to Gig0/0/0/2.100


### Example 1

In this example, the operator wants to move all configuration located under interface gig 0/0/0/0 to interface gig 0/0/0/2.  
In addition, all other references to the former interface (say in routing protocols, mpls lpd, etc) also need to be replaced with the new interface (gig 0/0/0/2)

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

Now the operator is ready to proceed with the change. The replace operation is run without using the "dry-run" keyword

```
RP/0/0/CPU0:iosxrv-1(config)#replace interface gigabitEthernet 0/0/0/0 with gigabitEthernet 0/0/0/2
Loading.
365 bytes parsed in 1 sec (357)bytes/sec
RP/0/0/CPU0:iosxrv-1(config)#

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
After committing the configuration we observe the new configuration and references to interface gig 0/0/0/2

```
RP/0/0/CPU0:iosxrv-1#show run
Thu May 26 07:02:54.045 UTC
Building configuration...
!! IOS XR Configuration 6.1.1.14I
!! Last configuration change at Thu May 26 07:02:48 2016 by cisco
!
hostname iosxrv-1
interface MgmtEth0/0/CPU0/0
 shutdown
!
interface GigabitEthernet0/0/0/1
 shutdown
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
In the previous example, both interfaces had the same configuration statements (description and IPv4 address). With the replace operation, the config from interface gig 0/0/0/1 was moved / merged with the config under interface gig 0/0/0/2

This example will cover the case where the original and new interfaces have "different" configuration statements and the user desires to only apply the config from the original interface and not the merge

We start with an initial config where interface gig 0/0/0/0 and gig 0/0/0/2 have different configuration statements (see non-default mtu on gig 0/0/0/2)  

```
RP/0/0/CPU0:iosxrv-1#show runn

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

To achieve the goal, the user destroys the interface gig 0/0/0/2 by using the "no" command
This is "immediately" followed by the replace operation

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

Observe that ...

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

In the second mode of operation, the user can provide string patterns

Note that the input is considered a regex string; e.g. replace pattern 'x' with 'y'  
So if you are trying to replace an IPv4 address such as 1.2.3.4, remember to escape the '.' as otherwise it would match any char  

```
Example: replace pattern '1.2.3.4' with '25.26.27.28' will match and replace both 1.2.3.4 and 10203040  
Example: replace pattern '1\.2\.3\.4' with '25.26.27.28' will match only 1.2.3.4 and not 10203040
```

xxx

```
RP/0/0/CPU0:PE1(config)#replace pattern ?
  regex-string  pattern to be replaced within single quotes
  
RP/0/0/CPU0:PE1(config)#replace pattern 'regex_1' with 'regex_2' ?
  dry-run  execute the command without loading the replace config
  <cr>  
```  

*  Renaming class-maps or flex-cli groups itself with replace may not go thru the commit due to classmap interdependecy on policymap etc.

*  Always use replace “dry-run” keyboard in order to validate changes that would be performed by the replace operation

## Example 3:

```
RP/0/0/CPU0:PE1(config)#replace interface gigabitEthernet 0/0/0/0 with gigabitEthernet 0/0/0/1 dry-run
RP/0/0/CPU0:PE1(config)#replace pattern '10\.20\.30\.40' with '100.200.250.225‘
RP/0/0/CPU0:PE1(config)#replace pattern 'GigabitEthernet0/1/0/([0-4])' with 'TenGigE0/3/0/\1'
```

xxx
