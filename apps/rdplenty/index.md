---
layout: bare
exclude: true
sitemap:
  exclude: 'yes'
title: "RDPlenty"
description: "Simple RDP Client for macOS"
date: 2026-09-17 00:00:00 -0700
---

RDPlenty is a streamlined RDP client for macOS. It lets you connect to Windows PCs and virtual machines using Microsoft's Remote Desktop Protocol, while keeping the experience simple, Mac-friendly and easy to integrate into larger workflows.

It deliberately avoids some of the complexity and troublesome features found in more general-purpose RDP clients, while adding automation features that are often missing elsewhere.

* TOC
{:toc}

## Features

- Connect to saved Windows PCs or make one-off RDP connections.
- The remote desktop is always scaled proportionally to fit the application window. There are no scrollbars to manage when the remote resolution and window size differ.
- Keyboard handling is designed around Mac muscle memory. The left ⌘ key acts as Ctrl by default, so normal Mac shortcuts such as ⌘S, ⌘F, ⌘A, ⌘C and ⌘V work naturally in Windows rather than only special-casing Copy and Paste.
- Clipboard sharing is either enabled in both directions or disabled entirely. There are no separate Mac-to-Windows and Windows-to-Mac clipboard modes to configure.
- Plain text, formatted text and images can be copied between the Mac and the remote machine.
- Remote audio playback is supported.
- No invasive permissions, accounts to register, telemetry, or cloud dependency.
- No broad system-control permissions — the app does not ask to "administer your computer."
- Credentials can be stored securely in the macOS Keychain.
- Certificates can be trusted for an individual saved connection without adding them to the system Keychain.

## Automation and workflow integration

RDPlenty is designed to work well as part of a larger workflow rather than only as a standalone GUI application.

A saved connection can run a local pre-connect script before RDP is started. This can be used to wake a sleeping computer, start a virtual machine, establish a VPN or perform other preparation required before the remote machine becomes reachable.

Connections can also be initiated through:

- a bundled command-line interface
- application URLs
- AppleScript / `osascript`

The command-line and scripting interfaces can start saved connections or make one-off connections directly to a host, with options such as account, resolution, full-screen mode, clipboard sharing and remote sound.

## Simpler by design

Some RDP features are deliberately not supported.

### File transfer

RDP drive redirection and file transfer through the clipboard are not supported.

These mechanisms can appear to work well for small transfers and then perform very poorly with large files or directory trees. Clipboard file transfer can also leave an RDP client tied up while a large transfer completes or fails.

For moving files between Mac and Windows, SMB, `scp` or cloud storage are generally better suited to the job.

### Printer redirection

Printer redirection is not supported.

Cross-platform RDP printer redirection depends on printer and driver compatibility between Windows and macOS and frequently does not work well. It is generally simpler to print to PDF, or transfer the document and print it locally from the Mac.

### Other features

The following features are not currently implemented and may be considered in the future if there is sufficient demand:

- RDP Gateway
- Smart card redirection
- Microphone redirection
- Camera redirection

## No accounts, no cloud service, no telemetry

RDPlenty does not require an account or any cloud service. It does not collect telemetry.

The only permission the app asks for is local network access. It does not request system-control permissions such as permission to "administer your computer."

Note that the consequence of having no telemetry is that the developer does not automatically learn about crashes, connection failures or other problems. If you encounter an issue, please report it.

## Support

Please report bugs and suggestions to <a id="maillink" data-mail="firmest.privets.34" data-domain="icloud.com" href="#">this e-mail</a>.

Bug reports, comments and suggestions, constructive or otherwise, are welcome.

If you are reporting a technical problem, the application's logs may contain useful diagnostic information. Please review them before sharing because logs can contain private information such as host names, account names, network addresses or connection details.

## Privacy Policy

RDPlenty **app** does not collect, store or share any of your data and does not track your usage in any way.

If you **email** the developer for support or other feedback, the emails and email addresses will be retained indefinitely. Email addresses will be used only to respond to the issues, questions or suggestions raised and will never be used for marketing.

This **website** (not the app!) uses Google Analytics to understand how visitors engage with it. This information will never be used for marketing.

Google Analytics' own privacy policy is available at [How Google uses information from sites or apps that use our services](https://policies.google.com/technologies/partner-sites).

You can prevent Google Analytics from recognizing you on return visits to this site by disabling cookies in your browser or by installing the [Google Analytics Opt-out Browser Add-on](https://tools.google.com/dlpage/gaoptout).

<script>
let maillink = document.getElementById('maillink')
maillink.innerHTML = maillink.getAttribute('data-mail') + '@' + maillink.getAttribute('data-domain')
maillink.href = 'mailto:' + maillink.getAttribute('data-mail') + '@' + maillink.getAttribute('data-domain') + '?subject=RDPlenty Issue'
</script>
