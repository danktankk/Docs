<h1>
  <img src="https://github.com/user-attachments/assets/b55803a8-a969-4b5a-b29e-7b982661d7ba"
       alt="mTLS Logo"
       width="250"
       align="middle" />
  Cloudflare mTLS Setup Guide
</h1>

## 1. Generate a Client Certificate

- Log in to the Cloudflare Dashboard.
- Navigate to **SSL/TLS > Client Certificates**.
- Click **Create Certificate**.
  - Choose **Private key type**: ECC.
    - *ECC offers stronger security per bit, better performance, and efficiency compared to RSA.*
  - Set **Certificate Validity** (default is 10 years).
- Click **Create**.
<img src="https://github.com/user-attachments/assets/7c71bb8e-83d8-46c0-a23d-70048098c876" alt="Create Client Certificate" width="600" height="auto">

- Copy and paste the `.crt` and `.key` contents into separate files using your CLI.
<img src="https://github.com/user-attachments/assets/784dd7b8-d31f-4724-acf5-c74c6ad4ae58" alt="Edit Hosts" width="300" height="auto">

## 2. Enable mTLS for Your Hostname

- In the same **Client Certificates** section, click **Edit** at the top.

<img src="https://github.com/user-attachments/assets/998fd77b-9fa9-40c2-a76f-cf0c770ac75a" alt="Add Hostnames" width="600" height="auto">

- Enter the hostname(s) you want to secure with mTLS:
  - e.g., `domain.com` and `*.domain.com`.

- Click **Save**.

## 3. Configure WAF Rule

- Navigate to **Security > WAF**.
- Click **Create Rule** and configure as follows:
<img src="https://github.com/user-attachments/assets/990e7083-6be2-4759-93fe-8ee7fdf0358b" alt="WAF Dashboard" width="600" height="auto">

  - **Rule Name**: `Block Unverified Clients`.
  - **Field**: `Client Certificate Verified`.
    - **Operator**: `equals`.
    - **Value**: `Off`.
  - **And Condition**:
    - **Field**: `Hostname`.
    - **Operator**: `contains`.
    - **Value**: `domain.com`.
  - **Action**: `Block`.
<img src="https://github.com/user-attachments/assets/3b495350-2a7d-41e7-9396-a3c87f8a1042" alt="WAF Rule Configuration" width="600" height="auto">

- Click **Deploy** to activate the rule.

## 4. (Optional) Adjust DNS Settings

- Navigate to **DNS**.
- Change the proxy status of your domain to **DNS Only**.
  - This can drastically improve load times for SearXNG.

## 5. Create a `.p12` Certificate Bundle

- Open a terminal in the directory you previously saved your mTLS key pair.
- Run the following command:

  ```bash
  openssl pkcs12 -export -out cf-ecc.p12 -inkey cf-ecc.key -in cf-ecc.crt
  ```

  - Replace `cf-ecc.key` and `cf-ecc.crt` with your actual key and certificate filenames.
  - You'll be prompted to set an export password.
  - The resulting `cf-ecc.p12` file can be used on browsers and mobile devices.

## 6. Install the `.p12` Certificate in Your Browser

### LibreWolf

<img src="https://github.com/user-attachments/assets/29d4f085-24e8-45af-8bfe-21013548b30c" alt="LibreWolf Certificate Settings" width="600" height="auto">

<img src="https://github.com/user-attachments/assets/13000556-8f70-4fb2-bffa-a9a9d2886111" alt="LibreWolf Certificate Import" width="600" height="auto">

### Brave

<img src="https://github.com/user-attachments/assets/33d8fedc-409c-4729-9978-23fc0dacddcc" alt="Brave Certificate Settings" width="600" height="auto">

<img src="https://github.com/user-attachments/assets/cc0c4c9d-0554-471c-87e5-4c877f747323" alt="Brave Certificate Import" width="600" height="auto">

### Chrome

<img src="https://github.com/user-attachments/assets/3ff40b38-1195-48f8-adb2-26eef008c402" alt="Chrome Certificate Settings" width="600" height="auto">

<img src="https://github.com/user-attachments/assets/22693678-ee44-4718-86be-f71caec6255b" alt="Chrome Certificate Import" width="600" height="auto">

### Android

Installing a `.p12` certificate on Android is pretty straightforward:

- Android certs are system-wide and are easy to find as well.  Go into settings for your device and search for `cert` and then tap on `VPN & app user certificate`.
- Select your .p12 file, input the password, and thats pretty much it.
- Not all browsers are made the same and due to this, some browsers dont work as well as others.  I know the chrome, opera, and edge work well.  Brave doesnt always work but has recently started showing improvement.

### Apple
- Dont care.

## 7. Test the mTLS Configuration

- Open your SearXNG instance.
- You should be prompted to select your certificate from a dropdown.
- Attempt to access the instance from a device without the certificate installed.
  - Access should be denied.
- Once confirmed, proceed to install the certificate on your other devices.
- Document the steps you've taken for future reference.

---

# **IMPORTANT:** 
### Always keep your `.p12` file and its associated password secure. If lost, you'll need to regenerate the certificate and repeat the installation process on all devices.



