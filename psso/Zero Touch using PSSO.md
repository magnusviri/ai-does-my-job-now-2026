This explains how to set up ADE Zero Touch using Apple's PSSO on macOS 26.

Note, this is using the device first approach, not the user first approach. The difference is that devicer first onboards to Jamf then authenticates to Entra. The user first approach autheticates to Entra befofe enrolling in Jamf.

Also note, this setup will set up PSSO on Macs that have already been enrolled. It works for both.

## Requirements

- Company Portal
- Config Profile
- Prestage Enrollment
- Inventory Preload

## Setup PSSO

### Company Portal

Get [CompanyPortal-Installer.pkg](https://go.microsoft.com/fwlink/?linkid=862280) and add it to Jamf.

### Config Profile

Download [ALL - PSSO MS Entra ID Kerberos.mobileconfig](ALL - PSSO MS Entra ID Kerberos.mobileconfig) and upload it. You can create this from scratch if you want. But it must have "Associated Domains" and 2 "Single Sign-on Extension" payloads, one payload is type "SSO", the other is "Kerberos".

If you have local admin accounts, add them to "Non-Platform SSO accounts" or else when you try to login with them, they will be forced to authenticate to Entra ID.

### Config Profile Scope

Install the config profile *after* the Company Portal is downloaded. The best way to do that is to create a Smart Group with this criteria:

"Application Title" "has" "Company Portal". You can also create an Extension Attribute that gets the Company Portal version and your smart group can check to see if it has a non-empty version, which is what I'm doing. This way, every computer that has Company Portal will be configured for PSSO.

### Company Portal Install Policy and Scope

Create a policy that installs Company Portal. Scope it to the computers where you want PSSO enabled.

This is all you need to do if you want to configure computers that have already been enrolled.

You should test this first to make sure it all works before moving to ADE.

## ADE 

If you want Zero Touch, proceed. Obviously you need ADE setup before you can do any of this. See [the Apple School Manager page](../Apple School Manager.md) for more info on that. This section will only discuss the changes you need to flip so you can do PSSO Zero Touch.

### Prestage Enrollment

#### General tab

- Check "Enable Simplified Setup for Platform Single Sign-on (macOS 26 or later)"
- Choose "Device first (enrollment completes befoer SSOe profile)"
- "Platform Single Sign-on Apple Bundle ID": "com.microsoft.CompanyPortalMac"

#### Account Settings tab

- You must create an account or else your first PSSO user will be admin, and you don't want that.
- If you want this account to be LAPS enabled go to "Settings", "Computer management", "Security". I don't recommend it because it is not as good as the LAPS account type that is created by "Settings", "User-initiated enrollment", "Computers". Note, even though it says, "User-initiated enrollment", the "Managed local administrator account" will still be created for Macs enrolled with ADE.

#### Config Profile tab

Check "ALL - PSSO MS Entra ID Kerberos".

Note, even though this is installed by Prestage, it will be removed just as quickly if the Mac does not have the config profile scoped to it. The only ways I know to do this is to either scope the config profile to all of your computers or to use Inventory Preload (see below). It's kind of a chicken-egg situation. There might be a better way to do this, but it works for me.

Note, you may also want to include a FileVault config profile, security tool config profiles, and maybe even UConnect.

#### Enrollment Packages tab

Add CompanyPortal-Installer.pkg.

### Inventory Preload

Adding everything above will get it all setup but it will fail unless you make sure that your Mac **keeps* the "ALL - PSSO MS Entra ID Kerberos" config profile once it enrolls, and it wont keep that profile unless you scope that profile to all computers or to a "department" that is set by Inventory Preload. I'm not going to explain Inventory Preload. Good luck.

So 2 things, create a department for "PSSO", then scope "ALL - PSSO MS Entra ID Kerberos" to that department.

Note, once your computer runs `jamf recon` after enrollment, if you created a smart group for "Company Portal", then that should be good enough to keep the config profile on the computer and you can change the department if you want. I know `jamf recon` runs at enrollment but in my experience, "ALL - PSSO MS Entra ID Kerberos" was almost always removed and caused the process to fail until I forced it on by using a department.

## Enrollment

After clicking the "Enroll" button at Setup Assistant, it will ask the user to authenticate to Entra. The Window title will be "Register your device with Microsoft Entra". Make sure the user enters, "account@example.com" (replacing "account" with their actual account name) and uses their account password.

Right after that dialog, a second window will appear with the heading "Sign In to Your Organization". Again, use "account@example.com" and the account password.

After setting the Time Zome, the user will be dropped to the login window password dialog. The user needs to enter "account@example.com" and their account password a **third** time. The first 2 times it was authenticating to Entra or something. This last time it is creating the account on the computer and finally logging in.

If you installed a FileVault config profile using Prestage Enrollment, the user will be prompted to encrypt the drive. The user account will get a Secure Token (if you're using FileVault).

All other logins after the first time can just be "account", not "account@example.com". Whew!

Also, remember that unless you create an account with the Prestage Enrollment, the PSSO account will be admin (see ablove).

## Other Accounts, Secure Tokens, and LAPS

It is possible to have 2 managed local administrator accounts. If you create both of these, you will have to use 2 different names, they can't be the same. These accounts will not get Secure Tokens unless **you** log them in. That means if you do nothing, only the PSSO end user will have the Secure Token and only that account will be able to decrypt the boot disk. Except Jamf does save the decryption keys, so it isn't the end of the world, you just have to click "Forgot password" at the loginwindow, boot to Recovery Mode and then reset a password. If you reset the password for a user without the Secure Token, resetting the password in Recovery Mode will give it the Secure Token.

The managed local administrator account created using "Settings", "User-initiated enrollment", "Computers" is called the "jamf binary" managed account. That's because the jamf binary can change that account password, so it's easier for Jamf to manage LAPS with this account.

The other managed local administrator account created with "Settings", "Computer management", "Security" is called the "Prestage" managed account. It can only be managed by MDM commands sent by the Jamf server. Because it can only be managed by MDM, if you use LAPS and FileVault (and give it a Secure Token), those 2 passwords will get out of sync. So don't use LAPS, FileVault, and the MDM managed account.

However, you must have the Prestage MDM managed account or else your first PSSO account will be admin. So maybe you don't care about it and you can just use LAPS with it anyway and let it get out of sync and who cares.

## References

- [Jamf docs](https://learn.jamf.com/r/en-US/jamf-school-documentation/Distributing_the_Company_Portal_App_from_Microsoft_to_End_Users)

