---
layout: page
exclude: true
title: Managing Windows networking components
---

{::options parse_block_html="true" /}

<div style="background-color:#f55c51;border-radius:10px;padding:10px">
<sm>This is an archive copy of an old FAQ for microsoft.public.win32.programmer.networks newsgroup I once wrote. People still follow links to it so I keep it alive. Note that it was written in 2005-2006 and information here may be obsolete.

Use at your own risk</sm>
</div>
<br>

* TOC
{:toc}
{: #toc}

[aaab]: https://www.betaarchive.com/wiki/index.php/Microsoft_KB_Archive/311272
[caaa]: https://learn.microsoft.com/en-us/windows/win32/api/lmserver/nf-lmserver-netservergetinfo
[caab]: https://learn.microsoft.com/en-us/windows/win32/api/lmwksta/nf-lmwksta-netwkstagetinfo
[caac]: https://learn.microsoft.com/en-us/windows/win32/netmgmt/network-management
[caaf]: https://learn.microsoft.com/en-us/windows/win32/api/lmserver/nf-lmserver-netserverenum
[cbaa]: https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-operatingsystem
[cbac]: https://learn.microsoft.com/en-us/windows/win32/wmisdk/wmi-start-page
[cbad]: https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-networkadapterconfiguration
[cbae]: https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-networkadapter
[ccaa]: https://learn.microsoft.com/en-us/windows/win32/wnet/windows-networking-wnet-
[ccab]: https://learn.microsoft.com/en-us/windows/win32/api/winnetwk/nf-winnetwk-wnetopenenumw
[ccac]: https://learn.microsoft.com/en-us/windows/win32/api/winnetwk/nf-winnetwk-wnetenumresourcew
[cfaa]: https://learn.microsoft.com/en-us/windows/win32/api/Iphlpapi/nf-iphlpapi-setifentry
[cfab]: https://learn.microsoft.com/en-us/windows/win32/iphlp/ip-helper-start-page
[cfac]: https://learn.microsoft.com/en-us/windows/win32/api/iphlpapi/nf-iphlpapi-getiftable
[cfad]: https://learn.microsoft.com/en-us/windows/win32/api/iphlpapi/nf-iphlpapi-getadaptersinfo
[cfae]: https://learn.microsoft.com/en-us/windows/win32/api/iphlpapi/nf-iphlpapi-getadaptersaddresses
[cfaf]: https://learn.microsoft.com/en-us/windows/win32/api/iphlpapi/nf-iphlpapi-getifentry
[cfag]: https://learn.microsoft.com/en-us/windows/win32/api/iphlpapi/nf-iphlpapi-sendarp
[cgaa]: https://learn.microsoft.com/en-us/previous-versions/windows/hardware/network/ff547694(v=vs.85)
[cgab]: https://learn.microsoft.com/en-us/previous-versions/windows/hardware/network/ff548975(v=vs.85)
[chaa]: https://learn.microsoft.com/en-us/windows/win32/api/_devinst/
[ciaa]: https://learn.microsoft.com/en-us/windows/win32/api/_ics/
[cjaa]: https://learn.microsoft.com/en-us/windows/win32/api/sensapi/nf-sensapi-isnetworkalive
[cjab]: https://learn.microsoft.com/en-us/windows/win32/api/sensapi/nf-sensapi-isdestinationreachablew
[ckaa]: https://learn.microsoft.com/en-us/windows/win32/api/ras/nf-ras-rasenumconnectionsw
[ckab]: https://learn.microsoft.com/en-us/windows/win32/rras/portal
[ckac]: https://learn.microsoft.com/en-us/windows/win32/api/ras/nf-ras-rasenumentriesw
[cmaa]: https://learn.microsoft.com/en-us/windows/win32/adsi/enumerating-adsi-objects
[cnaa]: https://learn.microsoft.com/en-us/windows/win32/api/wininet/nf-wininet-internetgetconnectedstate
[cnab]: https://learn.microsoft.com/en-us/windows/win32/api/wininet/nf-wininet-internetcheckconnectionw
[coaa]: https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-writeprivateprofilestringw
[cpaa]: https://learn.microsoft.com/en-us/windows/win32/snmp/simple-network-management-protocol-snmp-
[daaa]: https://learn.microsoft.com/en-us/windows/win32/api/_netbios
[faae]: http://tangentsoft.net/wskfaq/examples/getmac-netbios.html
[faaf]: http://tangentsoft.net/wskfaq/examples/getmac-snmp.html
[faag]: http://tangentsoft.net/wskfaq/examples/getmac-rpc.html
[iaaa]: http://groups.google.com/group/microsoft.public.win32.programmer.networks/msg/c137df21ad63a2e0
[iaac]: http://groups.google.com/group/microsoft.public.vb.winapi/msg/5506e4fc859a3f86

---

## Technologies

**Problem**

What technologies are available for managing Windows networking components and configuration?

**Solution** 

A whole lot of them. The following is a short list of most important ones with brief description of uses, advantages and disadvantages

* [IP Helper][cfab]<br>
  IP Helper is a plain Win32 API for manipulating TCP/IP configuration. It's main advantage is that it is just a normal Win32 API (no COM, no weird scripting stuff, no including DDK headers). However, it has quite a few disadvantages. First of all it is heavily oriented towards reading things (get functions) rather than modifying them (set functions). Though most common set operations are supported often you will need to use something else from this list. Second problem is that IP Helper is rather new API so its support on various Windows flavors is uneven. Some functions may not be available below XP and may behave differently. It's future is also uncertain. Microsoft seems to favor WMI (see below) so it is not sure whether IP Helper will be actively developed in the future.

* [WMI][cbac]<br>
  This is Microsoft's flagship configuration API. It is not designed exclusively for networking but rather is a whole framework for manipulating machine configuration. Similar to IP Helper it is mostly read-only API though some set methods are provided. Interestingly, WMI and IP Helper tend to complement each other for modifying stuff since things "settable" in one are read-only in another and vice versa. WMI's main advantages are ability to be called remotely, extensibility and support for multiple languages including scripting ones. Plus, since it is the recommended configuration API it's future looks bright. However, with these advantages comes the price. Expect a steep learning curve. Being familiar with COM and directory-like services such as AD or LDAP helps but this is not all. The framework is designed to be scripting friendly, which means that absolutely everything is done at runtime. This makes programming in strongly typed languages a pain. Then there is still the problem of things working somewhat differently on different platforms. Only because everything is done at runtime you will only learn about it when your application crashes. Expect to write once and debug everywhere as Java motto goes ;-). Personally I tend to avoid WMI unless absolutely necessary and from my online observations so do many other people.

* [Network Configuration Interfaces][cgaa]<br>
  For reasons that are beyond my imagination Microsoft had chosen to document this COM API in DDK making it virtually unknown to user-mode Win32 programmers. It a pity because this is a nice, classical COM API that allows manipulation of network hardware stack (i.e. everything below actual protocols). When you open property pages for a network connection you can see this API in action. It allows you to do manipulate bindings i.e. what protocols and services are enabled for a particular adapter, enable and disable adapters, open their property pages and more. The main advantage of this API is that it is very user-mode friendly gateway to hardware stuff. The only disadvantage is poor documentation and somewhat limited functionality.

* [Device Installation API][chaa]<br>
  Supported since Win2k and in a limited fashion on NT this API is for installation of hardware and drivers including networking ones. Some things are doable only through this API so you may occasionally need it.

* [Shell network interfaces][ciaa]<br>
  The shell network interfaces are an interesting thing. They are scantly documented in MSDN in conjunction with internet connection sharing (ICS) but they are very useful for many other tasks. Unlike other network management APIs these interfaces work with network connections, which are the Windows shell abstraction, rather than hardware adapters. Since network connections are the thing that normal user usually sees these interfaces fit nicely for usage in GUI driven code.

* [Net API][caac]<br>
  The network management functions have a dual role. First, they are the original (pre-Active Directory) way to manage user accounts and related information. Second they are the primary way to manage Windows (as opposed to general TCP/IP) network resources, such as servers, workstations and shares. It can also do some unexpected thing like schedule remote jobs or send messages to machines. Some of the functionality of this API is directly exposed to user via net command line utility.

* [WNet API][ccaa]<br>
  This API deals with adding and removing mapped drives or other Windows network connections in various and assorted ways. Compared with Net API its main advantage is that it adds common GUI stuff such as standard connection dialog boxes and messages.

* [RAS API][ckab]<br>
In Microsoft world Routing and Remote Access (RAS or RRAS) include not only conventional dial-up but also VPN and some more esoteric kinds of connections. This API allows to manage and manipulate them. Curiously wireless connections which are logically very similar are managed differently.

* Configuration files<br>
  Windows TCP/IP networking has its origins in Unix and retains some of the configuration files Unix uses. Take a look at your `%systemroot%\system32\drivers\etc` directory to see them.

* Manual registry modifications
  Sometimes, though much more rarely than people think, there is no other way than modify the registry. Use this only as a last resort since Microsoft usually makes no guarantees about how its configuration information is stored there.

[Back to top](#toc)

---


## Enabling and disabling interface/adapter
{: #enable}

**Problem**

I want to programmatically enable or disable a particular network interface/adapter/card.

**Solution**

There are multiple ways to do it depending on what exactly you need and on which OSes. Some of the possibilities are outlined below


<div style="margin-left:20px">

**Using [IP Helper][cfab]**

The API you need is [SetIfEntry][cfaa]. Note that this function "administratively" disables or enables the interface. This method is the most simple one but has a few problems. First, it _does not_ produce the same results as disabling the network adapter from Network Connections folder or device manager. Second, it only disables TCP/IP so you can still use Netbios and other low-level communication APIs. Thus it is usually unsuitable for security related tasks.

**Using [Device Installation API][chaa]**

_[Thanks to Arkady Frenkel for this tip]_ If you have DDK you can look at its sample enable (look in `src\setup\enable`) that demonstrates how to use it. In a [newsgroup post][iaaa] Arkady noted the following

> I heard from MS guys that some side effects take place with enable example but I used it for both network and serial drivers and for me that work like a charm.

If you don't have DDK, Alick Ye from Microsoft [posted equivalent code][iaac]. This method disables the actual device much like doing it manually from Device Manager. It is also supported on wider variety of platforms. Also you may want to take a look at [DevCon tool][aaab] that comes with source code.

**Using [Shell Network Interfaces][ciaa]**

This method is most useful when you want to enable or disable network connection which appear in Network Connections shell folder. Unfortunately it is only available for XP and higher. The following sample demonstrates how to use them to enable or disable a connection.

<details><summary markdown='span'>Click here to expand/collapse sample
</summary>

```cpp
#include <netcon.h>
// wszName is the name of the connection as appears in Network Connections folder
// set bEnable to true to enable and to false to disable
bool EnableConnection(LPCWSTR wszName, bool bEnable)
{
    bool result = false;
    typedef void (__stdcall * LPNcFreeNetconProperties)(NETCON_PROPERTIES* pProps);
    HMODULE hmod = LoadLibrary("netshell.dll");
    if (!hmod) 
        return false; 
    LPNcFreeNetconProperties NcFreeNetconProperties = 
        (LPNcFreeNetconProperties)GetProcAddress(hmod, "NcFreeNetconProperties"); 
    if (!NcFreeNetconProperties ) 
        return false; 

    INetConnectionManager * pMan = 0; 

    HRESULT hres = CoCreateInstance(CLSID_ConnectionManager, 
                                    0, 
                                    CLSCTX_ALL, 
                                    __uuidof(INetConnectionManager), 
                                    (void**)&pMan); 
    if (SUCCEEDED(hres)) 
    { 
        IEnumNetConnection * pEnum = 0;  
        hres = pMan->EnumConnections(NCME_DEFAULT, &pEnum); 
        if (SUCCEEDED(hres)) 
        { 
            INetConnection * pCon = 0; 
            ULONG count; 
            bool done = false; 
            while (pEnum->Next(1, &pCon, &count) == S_OK && !done) 
            { 
                NETCON_PROPERTIES * pProps = 0; 
                hres = pCon->GetProperties(&pProps); 
                if (SUCCEEDED(hres)) 
                { 
                    if (wcscmp(pProps->pszwName,wszName) == 0) 
                    { 
                        if (bEnable) 
                            result = (pCon->Connect() == S_OK); 
                        else 
                            result = (pCon->Disconnect() == S_OK); 
                        done = true; 
                    } 
                    NcFreeNetconProperties(pProps); 
                } 
                pCon->Release(); 
            } 
            pEnum->Release(); 
        } 
        pMan->Release(); 
    } 

    FreeLibrary(hmod); 
    return result; 
} 


int main() 
{ 
    CoInitialize(0); 
    EnableConnection(L"Local Area Connection", false); 
    CoUninitialize();
}
```
</details>
<br>
**Using [WMI][cbac]**

If your adapter uses DHCP it is possible to "disable" or "enable" it by releasing and renewing IP address using WMI. More details here. Of course this doesn't really disables or enables anything so I mention it here only as a trick in case somebody asks. There is no way to actually disable and adapter using WMI.

</div>

I should also mention here that it is impossible to use [Network Configuration Interfaces][cgaa] to enable or disable an adapter. Just in case you'd ask ;-)

[Back to top](#toc)

---

## Remote OS version
{: #version}

**Problem**

How can I determine Windows OS version and/or OS type on a remote computer?

**Solution**

Use either [NetServerGetInfo][caaa] with level 102 or [NetWkstaGetInfo][caab] with level 100.

Alternatively, you can use [WMI][cbac] and query [Win32_OperatingSystem][cbaa] class.

[Back to top](#toc)

---

## Connection to internet
{: internet}

**Problem**

How can I check whether my computer is connected to the Internet? Can I be notified when this happens? When my computer gets online I need to ...

**Solution**

I bet you are used to a dial-up connection 🙂. Internet in general is not an electric grid to which you are either connected or not. Think about a corporate user that can fire up his browser and visit Google but cannot access most other sites (because his gateway blocks them). Is he connected to Internet? Or take a home user that is behind a wireless router with a firewall. He can access any site but nobody can access his computer. Is he online? Most people would answer yes to these questions but for the purpose of **your application** this may not be so. What **you** probably care about is whether **your** application can connect to its server or be a server itself. In other words you actually want to know whether your application will be able to communicate, not whether "connection to the Internet" exists in some abstract sense.

After we have restated the question in this way it should be obvious that it is in general impossible to answer. Whether your application will be able to access another machine depends on many factors such as correct routing, security software allowing the traffic and the other machine being physically up. You cannot reliably determine all of this from the code that runs on your machine. <u>Thus the only way to be sure that you can communicate is to try it</u>. In other words just connect a socket (or whatever other communication mechanism you are using) and see if this succeeds. If it does you are all set and no other checks "whether Internet is available" are needed.

What if connection fails? The error code will rarely provide any useful information about the real reason for failure but many applications legitimately want to help the user with troubleshooting. To this end it could be helpful to distinguish between "server is down" and "you computer has no way to reach the server". As long as you remember that <u>there is no general solution</u> you can try some heuristics.

Arkady Frenkel had investigated some possible options in detail and kindly provided the following results

> **Network Line and Internet Connection Check FAQ**
> 
> Each function was tested on Windows 98, Windows NT SP6a, Windows 2000 , and Windows XP operating systems with a modem dialup and network card  connection (except for Windows XP where a modem was unavailable).
> 
> As seen below, the behavior of the connection testing APIs was much more predictable and stable in systems with a modem dialup connection.
> 
> All tests were done by unplugging the physical cable from the machine.
> 
> The [RasEnumConnections()][ckaa] API works well on all the systems, but is only available for testing modem connections.
> 
> The next set of APIs tested work for non-modem network connections as well:
> 
> 1. [IsNetworkAlive()][cjaa]
> 2. [InternetGetConnectedState()][cnaa]
> 3. [InternetCheckConnection()][cnab]
> 4. [IsDestinationReachable()][cjab]
>  
> **1) IsNetworkAlive()**<br>
> With a modem dialup connection, this API works as documented on
> all four OS's. NETWORK_ALIVE_LAN and NETWORK_ALIVE_WAN return TRUE if a connection exists, and
> FALSE if no connection.
>  
> With a network card connection, this API initially always returns TRUE regardless of the physical connection state for all OS's, but after a call to  InternetCheckConnection()
> returns FALSE (after waiting for a timeout - see below) calling IsNetworkAlive() correctly returns
> FALSE if the cable is unplugged.
> 
> **2) InternetGetConnectedState() :**<br>
> This API was tested with a verified, valid URL. With a modem dialup connection, the function works as documented, on all four OS's. 
> 
> With a network card connection, the function returns TRUE in all four OS's when plugged in.
> 
> With the cable unplugged the function continues to return TRUE for some time, but after a time-out it will
> return FALSE. When the cable is plugged back in the function will return TRUE only after a time-out period has elapsed.
> 
> On Windows 98, this function always
> returns TRUE, regardless of the state of the connection.
> 
> **3) InternetCheckConnection():**<br>
> With a modem dialup connection, this function works as documented in
> all four OS's. With a network card connection, when plugged in, it returns TRUE on Windows NT, Windows 2000, and Windows XP. 
> 
> On Windows 98 this function caused the test application to fault (under investigation).
>  
> On Windows NT, with the cable unplugged, the function returns FALSE (denoting an error) after a time-out period. GetLastError() returns 0, which is unusual.
>  
> On Windows 2000 this function behaves similarly to 
InternetGetConnectedState(); it will return the correct value after a time-out period.
> 
> **4) IsDestinationReachable() :**<br>
> This function is only available on Windows ME and Windows 2000.
>  
> With a modem dialup connection, this function works as documented. With a  network card connection, this function returns TRUE when the cable is plugged in, and returns FALSE when the cable is unplugged. Unfortunately, after a FALSE return, GetLastError() returns 0.
> 
> The last test was with the IP Helper APi GetIfEntry(), which returns the status of a adapter adapter. 
> 
> On Windows 98, Windows NT, and Windows 2000, using a modem dialup connection, GetIfEntry() sets the dwAdminStatus and dwOperStatus members of the MIB_IFROW structure equal to MIB_IF_OPER_STATUS_UNREACHABLE. 
>  
> On Windows XP ( the only OS with a network card connection tested),  the dwOperStatus member was set to MIB_IF_OPER_STATUS_OPERATIONAL  when network cable was plugged in, and set to MIB_IF_OPER_STATUS_NON_OPERATIONAL if the cable was unplugged.
> dwAdminStatus was set to MIB_IF_OPER_STATUS_UNREACHABLE in both cases.
>  
> On Windows XP it is possible to check the status of a connection very easily by using GetIfEntry().
> 
> On Windows 2000 and Windows XP, it may be possible to check the connection state via WMI. 
>  
> Another possibility for connection status checking is to use DeviceIOControl() to query the driver about the connection state.
>  
> Arkady Frenkel , Microsoft MVP [Windows SDK/Networking]


Some other things that you may want to try

* Check operational status of all interfaces using GetIfTable. At least one should be up.
* Check all the network adapters and validate their IP addresses, DNS servers and gateways. The [GetAdaptersInfo][cfad] API will help with that. If all the IP addresses are 0.0.0.0, DNS servers are not set etc. most likely the user is not physically connected.

[Back to top](#toc)

---

## Enumerate computers
{: #enum_computers}

**Problem**

How to find out all computer names on my network?

**Solution**

The answer depends on what you mean by "all computers" and "my network". In the most general case there is no solution. However, if what you really need is all Windows machines and by network you mean Windows domain or workgroup then it is certainly possible. Some ways to accomplish this are

* Use [NetServerEnum][caaf] API
* Use [WNetOpenEnum][ccab] and [WNetEnumResource][ccac] APIs
* [Enumerate][cmaa] Active Directory objects.

[Back to top](#toc)

---

## Advanced RAS/VPN Properties
{: #adv_ras}

**Problem**

I am trying to programmatically create RAS/VPN connection and some of the properties available through Windows GUI are not exposed by RAS API. How to specify them?

**Solution**

There are couple of options. Some things can be managed through WMI. The [Win32_NetworkAdapterConfiguration][cbad] class exposes a number of methods that allow you to change network adapter parameters. When WMI provides the necessary functionality it is always preferable as this is the recommended and supported API. Unfortunately sometimes WMI is not an option or it doesn't provide access to a specific setting.

Given that on NT-based systems information about RAS/VPN connection is stored in phonebook files, one possible solution is to directly modify them. The [RasEnumEntries][ckac] API allows to find the name of the phonebook file for each RAS entry. The default location of "all users" phonebook file is
`%ALLUSERSPROFILE%\Application Data\Microsoft\Network\Connections\Pbk\rasphone.pbk`
The format of a phonebook file is obviously the standard Windows INI file format which can be manipulated by [WritePrivateProfileString][coaa] and related functions. Each RAS/VPN connection is a section while its settings are key/value pairs. Most of the options have self-explanatory names and you always can find which option corresponds to a given GUI setting by modifying the connection properties in GUI and observing the changes in a phonebook file. The following table summarizes some options frequently asked about on Usenet. Note that these options are not documented or supported by MS so they may disappear or change meaning in the future. In addition I made my tests on XP SP2 and it is entirely possible that some of them may be missing or have different menaing on other Windows versions.

| Option                     | GUI Analog                                      |Possible Values  | WMI alternative
|----------------------------------------------------------------
| `ShowMonitorIconInTaskBar` | Show Icon in notification area when connected   |Boolean: 1 or 0  | Unknown
| `IpDnsFlags`               | Various setting on DNS tab of TCP/IP properties |Bitmask: 1 - Register this connection's address in DNS | `Win32_NetworkAdapterConfiguration.SetDynamicDNSRegistration`


[Back to top](#toc)

---

## MAC address
{: #mac}

**Problem**

How to find network card's MAC address?

**Solution**

There are multiple ways to do it depending on what exactly you need and on which OSes. Some of the possibilities are outlined below. Note that my intent is to show methods that are known to be <u>reliable</u> in a sense that they always produce unambiguous results. For completeness I also include unreliable methods at the end of this list but please do not use them.

<div style="margin-left:20px">

**Using [SNMP][cpaa]**

This method is described at [Winsock Programmer's FAQ][faaf]. Despite being in that FAQ for a long time it is relatively little known. Its biggest advantage is that this method is the most portable one and does not require any special machine configuration.

**Using [IPHelper][cfab]**

The MAC address can be obtained simply by calling [GetAdaptersInfo][cfad] (replaced on newer systems by [GetAdaptersAddresses][cfae]), [GetIfTable][cfac] or [GetIfEntry][cfaf]. If you can use it it is the simplest documented method. It is not available on older 9x systems though.

**Using [WMI][cbac]**

The class you need are [Win32_NetworkAdapter][cbae]. Alternatively you can use under-documented `MSNdis_EthernetCurrentAddress` class. Use the `root\WMI` namespace and examine the `MSNdis` class for properties of the class `MSNdis_EthernetCurrentAddress`.

**Sending ARP request**

This method is usually used to find MAC address of a remote machine but can also be used with a local one. The relevant API is [SendARP][cfag]

**Talking to a driver**

The query is made using the [IOCTL_NDIS_QUERY_GLOBAL_STATS][cgab] DeviceIoControl call.

The Windows NT DDK MACADDR sample shows how to make this call. 

**Using [Netbios][daaa] API**

<span style="color:red">This method is unreliable</span>. It is described in detail at [Winsock Programmer's FAQ][faae] and also in Knowledge Base article Q118623. Note that Netbios is being gradually phased out and may not even be present on modern machines.

**Using COM/OLE/RPC UUIDs**

<span style="color:red">This method is unreliable and can only be described as a hack</span>. More details can be found at [Winsock Programmer's FAQ][faag].

</div>

[Back to top](#toc)

---
