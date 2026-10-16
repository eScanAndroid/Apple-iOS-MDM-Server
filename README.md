# Renewal of the Apple Push Notification Service (APNs) Certificate

Step-by-step procedure for renewing the APNs push certificate used by the MDM server.

## Contents

- [1. Overview](#1-overview)
- [2. Before you start](#2-before-you-start)
- [3. Renewal steps](#3-renewal-steps)
  - [Step 1: Create the MDM Vendor CSR](#step-1-create-the-mdm-vendor-csr)
  - [Step 2: Upload the CSR to Apple](#step-2-upload-the-csr-to-apple)
  - [Step 3: Export the MDM private key](#step-3-export-the-mdm-private-key)
  - [Step 4: Create the push certificate CSR](#step-4-create-the-push-certificate-csr)
  - [Step 5: Extract the keys and certificate](#step-5-extract-the-keys-and-certificate)
  - [Step 6: Create applepush.csr](#step-6-create-applepushcsr)
  - [Step 7: Renew the push certificate with Apple](#step-7-renew-the-push-certificate-with-apple)
  - [Step 8: Prepare the final push certificate](#step-8-prepare-the-final-push-certificate)

---

## 1. Overview

The APNs certificate lets the MDM server communicate with Apple devices. It is used to push policies, commands and anti-theft instructions. The certificate is valid for **1 year** and must be renewed **at least 1 month before it expires**. Renew the existing entry rather than creating a new one, otherwise enrolled devices will have to be re-enrolled.

Two different certificates are involved in the process. They are easy to confuse:

| Certificate | Where it comes from | What it is used for |
|---|---|---|
| **MDM Vendor certificate** (`mdmvendor.cer`, `private.key`, `mdm.cer`) | Apple Developer portal, Certificates, MDM CSR | Only to sign `push.certSigningRequest` and produce `applepush.csr`. The MDM server never uses it to send pushes. |
| **Push certificate** (`mdm.p12`, `PushCert.pem`) | Apple Push Certificates Portal ([identity.apple.com/pushcert](https://identity.apple.com/pushcert/)) | Used by the MDM server to authenticate to APNs and send MDM pushes. This is the certificate that expires after a year. |

## 2. Before you start

- A Mac with **Keychain Access**, used to create the CSRs and export the certificates.
- The **same Apple ID** that was used to create the existing push certificate. A certificate created under one Apple ID cannot be renewed from another.
- An Apple Developer account with the **MDM CSR** option enabled. If it is disabled, email devprograms@apple.com and ask for the MDM CSR option to be enabled for creating an MDM Vendor certificate (Apple normally replies within one business day).
- **OpenSSL** on the machine where you run the commands. On Windows, Git for Windows includes one in `C:\Program Files\Git\usr\bin`; add that folder to PATH.
- **Python 2.7.18** and the signing script `mdm_vendor_sign.py` (see [Step 6](#step-6-create-applepushcsr)). The script in the original GitHub repository is written for Python 2.
- A single working folder (for example `MDM`) to hold every file created below.

## 3. Renewal steps

### Step 1: Create the MDM Vendor CSR

1. Open Keychain Access, then choose **Keychain Access → Certificate Assistant → Request a Certificate From a Certificate Authority**.
2. Enter the Apple Developer account email and a common name.
3. Select **Saved to disk** and save the CSR in the working folder.

### Step 2: Upload the CSR to Apple

1. Open the Apple Certificates, Identifiers & Profiles page and select **MDM CSR** under Production.
2. Continue through Apple's description of how to create a CSR (one already exists), then upload the `.certSigningRequest` file from Step 1 and click **Generate**.
3. Download the `.cer` file and name it **`mdmvendor.cer`**. It can also be downloaded later from the Certificates list.

> **Note:** `mdmvendor.cer` is already in DER format. If the conversion in Step 5 gives you trouble, this file can be used directly as the `--mdm` argument in Step 6.

### Step 3: Export the MDM private key

1. Open **`mdmvendor.cer`** in Keychain Access and select **Certificates** on the left.
2. Find **MDM Vendor: \<Common Name\>** and expand it using the arrow to reveal the private key.
3. Right-click the private key, choose **Export**, and save it as `private.p12`. Remember the export password if you set one.

### Step 4: Create the push certificate CSR

1. In Keychain Access, choose **Certificate Assistant → Request a Certificate From a Certificate Authority** again.
2. Enter an email (it may differ from Step 1) and a common name, select **Saved to disk**, and save the file as **`push.certSigningRequest`**.

> **Important:** Keep this Mac and its keychain. The private key created here is the one needed to export the final push certificate in Step 8.

### Step 5: Extract the keys and certificate

Open a terminal in the working folder with root access and run the commands below, in order.

```bash
openssl pkcs12 -in private.p12 -nocerts -out key.pem -legacy
openssl rsa -in key.pem -out private.key
openssl pkcs12 -in private.p12 -clcerts -nokeys -out cert.pem -legacy
openssl x509 -in cert.pem -inform PEM -out mdm.cer -outform DER
```

- The first command asks for the `private.p12` import password (if any) and then a new PEM pass phrase. Enter that same pass phrase when the second command asks for it.
- The third command asks for the `private.p12` import password again.
- `-legacy` is needed on both `pkcs12` commands with OpenSSL 3, because Keychain exports use an older encryption algorithm. If your OpenSSL version rejects the flag (older OpenSSL 1.x or the LibreSSL that ships with macOS), simply leave it out.

> **Warning:** the last command must use `-outform DER`. `DES` (as it appears in some older guides) is a cipher name, not a certificate format. It produces a file the signing script cannot read, and the script then fails with `Could not find certificate from <stdin>`.

Verify the result before continuing. It should print a subject starting with `MDM Vendor`:

```bash
openssl x509 -inform der -in mdm.cer -noout -subject
```

At the end of this step the folder holds `key.pem`, `private.key`, `cert.pem` and `mdm.cer`.

### Step 6: Create applepush.csr

1. Clone the repository:

   ```bash
   git clone https://github.com/eScanAndroid/Apple-iOS-MDM-Server.git
   ```

2. Copy **`push.certSigningRequest`**, **`private.key`** and **`mdm.cer`** into the repository's `vendor` folder.

Run the script from the `vendor` folder:

```bash
python mdm_vendor_sign.py --key private.key --csr push.certSigningRequest --mdm mdm.cer --out applepush.csr
```

Each of the three verification lines (`push.certSigningRequest`, `private.key`, `mdm.cer`) must print `OK`. When the script finishes it prints `DONE` and creates **`applepush.csr`**.

> **If the `mdm.cer` check fails:** run the verify command from Step 5. If it prints nothing useful, the file is not valid DER. Re-run the conversion with `-outform DER`, or use the `mdmvendor.cer` downloaded from Apple in Step 2 instead.

### Step 7: Renew the push certificate with Apple

1. Sign in at <https://identity.apple.com/pushcert/> with the **same Apple ID** used for the existing certificate.
2. Select the existing APNs certificate in the table and click **Renew**. Do not click *Create a Certificate*, which would issue a new certificate with a different topic.
3. Upload `applepush.csr` and download the resulting certificate.

### Step 8: Prepare the final push certificate

1. Open the downloaded certificate in Keychain Access on the Mac used in Step 4. In **My Certificates** it is listed as `APSP:<uuid>`, with the private key nested under it.
2. Right-click it and choose **Export**, saving the certificate together with its private key as `mdm.p12`.
3. Convert it to PEM for the MDM server:

   ```bash
   openssl pkcs12 -in mdm.p12 -out PushCert.pem -nodes
   ```

   Add `-legacy` to this command if OpenSSL reports an unsupported algorithm.

4. Convert the CA certificate (the latest SSL CA bundle file, `ca.cer`) to PEM for the MDM server:

   ```bash
   openssl x509 -inform DER -in ca.cer -out cacert.pem
   ```

5. Copy **`PushCert.pem`** and **`cacert.pem`** to the MDM server folder and restart or reload the MDM service so it picks up the new files.

Check the new certificate. The expiry date should be about one year from today, and the UID (the MDM topic) should be unchanged from the previous certificate:

```bash
openssl x509 -in PushCert.pem -noout -subject -enddate
```

> **Note:** if the private key does not appear under the certificate in Keychain Access, `push.certSigningRequest` was created on a different Mac or keychain. Export from that machine instead.

**Renewal complete.**
"# Apple-iOS-MDM-Server" 
