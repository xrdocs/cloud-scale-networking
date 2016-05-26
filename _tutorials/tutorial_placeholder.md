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
This feature allows for easy manipulation of router configuration
>
*  Ever wanted the ability to easily move configuration from one interface to another?
*  Or, rather wanted to change all repetitions of a given pattern in your router configuration?
>
If so, keep on reading ...
{: .notice--info}

```
RP/0/0/CPU0:PE1#configure
RP/0/0/CPU0:PE1(config)#replace ?
  interface  replace configuration for an interface
  pattern    replace a string pattern in configuration
```

xxx

```
RP/0/0/CPU0:PE1(config)#replace interface <ifid_1> with <ifid_2> ?
  dry-run  execute the command without loading the replace config
  <cr>
```

xxx

```
RP/0/0/CPU0:PE1(config)#replace pattern ?
  regex-string  pattern to be replaced within single quotes
  
RP/0/0/CPU0:PE1(config)#replace pattern 'regex_1' with 'regex_2' ?
  dry-run  execute the command without loading the replace config
  <cr>  
```

```
RP/0/0/CPU0:PE1(config)#replace interface gigabitEthernet 0/0/0/0 with gigabitEthernet 0/0/0/1 dry-run
RP/0/0/CPU0:PE1(config)#replace pattern '10\.20\.30\.40' with '100.200.250.225‘
RP/0/0/CPU0:PE1(config)#replace pattern 'GigabitEthernet0/1/0/([0-4])' with 'TenGigE0/3/0/\1'
```

xxx

## Some Caveats and Considerations

*  Replace interface X with Y causes sub-interfaces X.0, X.1, etc to also be replaced.
*    Example: replace Gig0/0/0/1 with Gig0/0/0/11 will cause sub-interface Gig0/0/0/1.100 to be replaced to Gig0/0/0/11.100

*  For pattern replace, the input is considered a regex string; e.g. replace pattern 'x' with 'y'
So if you are trying to replace 1.2.3.4 remember to escape the '.' as otherwise it would match any char
Example: replace pattern '1.2.3.4' with '25.26.27.28' will match and replace both 1.2.3.4 and 10203040.
Example: replace pattern '1\.2\.3\.4' with '25.26.27.28' will match only 1.2.3.4 and not 10203040

*  Renaming class-maps or groups itself with replace may not go thru the commit due to classmap interdependecy on policymap etc.

*  Always use replace “dry-run” keyboard in order to validate changes that would be performed by the replace operation




