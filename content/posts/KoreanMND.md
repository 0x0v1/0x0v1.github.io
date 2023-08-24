---
title: "Korean Military of Defence's Mobile Defence Security application"
date: 2023-08-24T12:52:12+01:00
draft: false
---

## The Korean Military of Defence's "Mobile Defence Security" application (국방모바일보안)

The Korean Military has a mandatory phone application for all soldiers. This application is used to support military personnel, staff and contractors to securely enter buildings and reduce the risk of data leaks. At the start of the year,  I found two vulnerabilities in this application that would leak GPS locations of user. The Korean Ministry of Defense refused to respond to me or my findings, despite patching the product and speaking to media about it.

The Military application called "국방모바일보안" (Defense Mobile Security) app has three different versions. They are for soldiers (employees), staff (civil servants) & external contractors. They applications are intended to use NFC and Bluetooth to allow enter military sites. The application was paid for by Korean tax payers, under funding from the Korean Ministry of Defense. The application and it's NFC/Bluetooth stations is rumored to have costed somewhere in the region of 30 Billion Korean Won, which is over 20million USD (though I should say, these sources are not vigorous). Though, it wouldn't surprise me, since most of the work was done by a private sector organization called MarkAny (https://www.markany.com).

The vulnerabilities I found were trivial, really. Simply, the applications for staff (civil servants) and external contractors had a broadcast receiver exported with no permissions. In simple terms, this means that any application outside (on the device) of the military app can call this receiver. When the broadcast receiver was called from outside of the application, the internal applications log file was exported to the devices SD card. This meant that any attacker of an Android device, could export the applications internal log file to the SD. Generally this is poor practice, especially when the log file contains sensitive data. Which it did. When the user would unlock a device, the GPS location would be stored in the log file. With coarse location and precise timestamps. This results in a privacy and security risk for those users. Evermore so, since their frequent and regular locations can be leaked this risks the security of military personnel - which in my opinion defeats the purpose of what the application was set out to do in the first place. 

I responsibly disclosed this vulnerability, detailing all my findings and remediation steps to the Korean Ministry of Defense. After 2 months of waiting. I heard nothing back from them. We tried multiple avenues and they did not reply. We ended up publishing my findings publicly (https://interlab.or.kr/archives/19264) and contacted multiple media outlets in Korea to demonstrate that this publicly funded application lacked privacy. We still heard nothing.

After multiple media outlets reported on my findings, we finally saw a Ministry of Defense spokesperson finally give a response to my findings ( https://www.boannews.com/media/view.asp?idx=118437). The spokes person acknowledged my finding said the following: *"We have also confirmed that two vulnerabilities have been found in apps for employees and outsiders that  allow attackers to extract personal information without requesting  permission, and we are working with the developer to compensate for the  vulnerabilities."*

On July 18th, the Ministry of National Defense (MND) released an update for the application which patched the vulnerabilities I found. 

In talking with people in the MND circles, we are told that the MND mostly shed blame onto the private sector developer of the application MarkAny (who's slogan is ironically "Protect what matters"). Despite this, we should acknowledge that this application was funded by tax payers in Korea. So whomever the developer is, it doesn't matter. Fundamentally the MND failed to properly secure a mandatory application for its military personnel and defied the purpose of the application in the first place. My view is this, any time an application is funded by tax payers, the code should be open source (https://publiccode.eu/en/).

The last point I want to add is that the MND have not communicated with myself or Interlab once in this whole process. Despite the fact that they acknowledged and patched my findings, they have yet to speak to us or thank us. I think that speaks volumes. 

Ovi