# TryHackMe: Checkmate Write-up

**Date:** June 19, 2026  
**Category:** Web Application Security / Password Cracking  

## Overview

**Checkmate** is a multi-level web challenge hosted on port `5000`. Each level presents a scenario with specific clues regarding password habits, company keywords, or personal details. Following these clues allows you to generate targeted wordlists, crack hashes, and retrieve the password required to unlock each stage and capture the flags.

> **Note:** Blind brute-forcing against the main application on port `5000` is out of scope and triggers a temporary cooldown.

---

## Level 1: Default Credentials

### Hint
Marco deployed a firewall at `firewall.thm:5001` and left the default credentials active.

### Walkthrough
1. Navigate to `http://firewall.thm:5001`.
2. Inspecting the login form reveals a placeholder suggesting `admin` as the username.
3. Testing common default passwords against `admin` quickly yields access.

* **Username:** `admin`
* **Password:** `12345`

---

## Level 2: Company Keyword Spidering

### Hint
The password is a company-related keyword (7 letters in length) found on the target web application (`port 5002`).

### Walkthrough
Instead of manually scraping the site for keywords, we can use **CeWL** to spider the web page and generate a custom wordlist filtered to a minimum length of 7 characters:

```bash
cewl -d 2 -m 7 --lowercase --with-numbers -w cewl_words.txt http://<TARGET_IP>:5002
```

Once the wordlist is generated, brute-force the login form using **ffuf** (or **Hydra**):

```bash
ffuf -u http://<TARGET_IP>:5002/login -X POST -d "username=admin&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -w cewl_words.txt -fc 401
```

* **Password:** `excellence`

---

## Level 3: OSINT & Targeted Wordlist Profiling

### Hint
Use personal details discovered in the previous stages to determine the user's password.

### Walkthrough
Using **CUPP (Common User Passwords Profiler)**, we can generate a tailored wordlist based on Marco's personal details (first name, last name, nickname, birth year, pet's name, etc.).

1. Clone and run CUPP in interactive mode:
   ```bash
   git clone [https://github.com/Mebus/cupp.git](https://github.com/Mebus/cupp.git)
   cd cupp
   python3 cupp.py -i
   ```
2. Answer the interactive prompts using the OSINT gathered about Marco Bianchi.
3. Use `ffuf` with the resulting custom wordlist to brute-force the login portal.

* **Password:** `bianchi2495`

---

## Level 4: SHA-256 Hash Cracking

### Hint
On `social.thm:5003`, uploaded profile pictures are automatically renamed to the SHA-256 hash of their original filename (`<SHA256>.png`). Identify the original filename of Marco's uploaded profile picture.




### Walkthrough
1. Right-click Marco's profile picture on `social.thm:5003` and copy the image URL/filename to extract the SHA-256 hash string.
2. Save the hash into a local file:
   ```bash
   echo "<EXTRACTED_SHA256_HASH>" > hash.txt
   ```
3. Run **Hashcat** in raw SHA-256 mode (`-m 1400`) using a dictionary attack (`-a 0`) against `rockyou.txt`:
   ```bash
   hashcat -m 1400 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
   ```

*(Submit the cracked plaintext filename without the `.png` extension to complete the level.)*

---

## Level 5: Pattern-Based SSH Brute-Force

### Hint
In Level 3, Marco left a post describing his exact password creation formula:
> *"My tip for strong password: I take a **company keyword**, **capitalize** it, then append the **year** like 2024 or any other number and an **exclamation mark**."*



### Walkthrough
While we could mutate the entire company keyword list from Level 2, Marco's post explicitly highlights `security` as the first keyword and mentions `2024` as an example year. We can use **Crunch** to quickly generate all 13-character combinations matching `Security20XX!`:

```bash
crunch 13 13 -t Security20%%! -o pass_marco.txt
```

Next, use **Hydra** to brute-force the SSH service using the generated wordlist:

```bash
hydra -l <USERNAME> -P pass_marco.txt ssh://<TARGET_IP>
```

*(Note: If this targeted pattern hadn't succeeded, the next step would be combining all capitalized keywords from `cewl_words.txt` with 4-digit numbers and a trailing `!`.)*

* **Password:** `Security2024!`
