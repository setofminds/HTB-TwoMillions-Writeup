# HTB-TwoMillions-Writeup

<h2>🔍 Step 1: Scanning the Target with Nmap</h2>

<p>The first step in any penetration test or CTF challenge is <strong>reconnaissance</strong> — discovering which services are running and potentially vulnerable.</p>

<p>I used <a href="https://nmap.org" target="_blank"><code>nmap</code></a> to scan the target machine at <code>10.10.11.221</code>:</p>

<pre><code>nmap -sC -sV -oN nmap_initial.txt 10.10.11.221</code></pre>

<h3>🛠️ What Do These Flags Mean?</h3>
<ul>
  <li><code>-sC</code> → Runs default scripts (basic detection and enumeration)</li>
  <li><code>-sV</code> → Attempts to identify the version of services</li>
  <li><code>-oN</code> → Outputs the scan result to a file (<code>nmap_initial.txt</code>)</li>
</ul>

<h3>🔎 Scan Results:</h3>
<img width="749" height="232" alt="image" src="https://github.com/user-attachments/assets/a992e7b6-da83-46db-9a5f-fce1b6b66af3" />


<h3>📌 Summary:</h3>
<ul>
  <li><strong>SSH (Port 22)</strong> is open — might be useful later for direct access via credentials.</li>
  <li><strong>HTTP (Port 80)</strong> is open — the primary entry point. We’ll focus our attack here first.</li>
</ul>

<p>➡️ <em>Let’s move on to web enumeration and explore the service running on port 80.</em></p>



<br>
<br>
<h2>🌐 Step 2: Accessing the Web Application on Port 80</h2>

<p>The <code>nmap</code> scan showed that port <strong>80/tcp</strong> is open and serving HTTP content using <strong>nginx</strong>:</p>

<pre><code>
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
</code></pre>

<p>From this, we can tell that the site is redirecting to a hostname: <code>http://2million.htb</code>, but your system doesn’t know how to resolve this domain yet.</p>

<h3>🛠️ Fixing the Redirect: Editing <code>/etc/hosts</code></h3>

<p>To resolve the domain name locally, we add an entry in the <code>/etc/hosts</code> file, pointing <code>2million.htb</code> to the machine’s IP address:</p>

<pre><code>sudo nano /etc/hosts</code></pre>

<p>Add this line at the bottom:</p>

<pre><code>10.10.11.221  2million.htb</code></pre>

<h3>✅ Result:</h3>
<p>Now, when you visit <a href="http://2million.htb" target="_blank">http://2million.htb</a> in your browser, the domain resolves properly, and you are greeted by the web interface of the machine. 🎉</p>

<img width="1363" height="554" alt="image" src="https://github.com/user-attachments/assets/0f3b4238-8afb-4e10-9dca-be4da9859e5a" />



<br><br><br>

<h2>🔎 Step 3: Directory Enumeration & Manual Exploration</h2>

<p>After getting access to the web application at <code>http://2million.htb</code>, the next logical step is to perform a directory or content discovery scan to find hidden paths.</p>

<h3>🚫 Gobuster Failed — Switched to Feroxbuster</h3>
<p>I usually use <a href="https://github.com/OJ/gobuster" target="_blank"><code>gobuster</code></a>, but it didn’t handle this site well (likely due to redirects or response handling issues). So, I switched to <a href="https://github.com/epi052/feroxbuster" target="_blank"><code>feroxbuster</code></a>:</p>

<pre><code>feroxbuster -u http://2million.htb -w /usr/share/seclists/Discovery/Web-Content/common.txt -o ferox.txt</code></pre>

<p>Unfortunately, the scan didn’t yield any interesting or accessible endpoints. That’s when I decided to switch strategies.</p>

<h3>🔍 Manual Inspection Pays Off</h3>
<p>While browsing the website manually and reviewing the page source (<code>Ctrl + U</code> or <code>Right Click → View Page Source</code>), I discovered a reference to a hidden route:</p>

<pre><code>/invite</code></pre>

<p>This page is not linked directly from the homepage, and it was not found by automated tools — showing the value of manual recon!</p>

<p>➡️ Visiting <code>http://2million.htb/invite</code> brought me to a suspicious-looking interface that seemed to mimic Hack The Box’s old invite code system. 🧩</p>
<img width="1365" height="514" alt="image" src="https://github.com/user-attachments/assets/c3325a6e-97c3-478f-bd33-77b7d1741dce" />

<br><br><br>
<h2>🕵️ Step 4: Inspecting Network Traffic & Discovering JavaScript Files</h2>

<p>To understand how the <code>/invite</code> system works, I opened the browser’s Developer Tools and navigated to the <strong>Network</strong> tab. Then I refreshed the page to observe all network activity:</p>

<ol>
  <li><strong>Right-click</strong> on the page → <code>Inspect</code></li>
  <li>Go to the <strong>Network</strong> tab</li>
  <li><strong>Refresh</strong> the page (F5 or Ctrl+R)</li>
</ol>

<img width="846" height="239" alt="image" src="https://github.com/user-attachments/assets/c2bb81f7-9dd0-4466-92f5-70c6e750e645" />

<p>Among the files loaded, I noticed a minified JavaScript file:</p>

<pre><code>http://2million.htb/js/inviteapi.min.js</code></pre>

<p>This file seemed responsible for handling the logic behind generating and validating invite codes. I clicked on it to inspect its contents and saw it was <strong>obfuscated</strong> or <strong>minified</strong> — hard to read at first glance.</p>

<br><br><br>
<h2>🧬 Step 5: JavaScript Deobfuscation & Invite Code Generation</h2>

<p>Once I identified the <code>inviteapi.min.js</code> file in the browser's Network tab, I opened it and saw that it was heavily <strong>obfuscated</strong> using a JavaScript <code>eval</code> wrapper and function encoding logic.</p>

<img width="1365" height="170" alt="image" src="https://github.com/user-attachments/assets/e97de263-fb41-490a-b200-a8d27ad76283" />


<p>To analyze it properly, I copied the JS code into an online beautifier: 
<a href="https://lelinhtinh.github.io/de4js/" target="_blank">de4js</a>. After beautifying, the code revealed two key functions:</p>

<pre><code>
function verifyInviteCode(code) {
    var formData = { "code": code };
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',
        success: function(response) { console.log(response) },
        error: function(response) { console.log(response) }
    });
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',
        success: function(response) { console.log(response) },
        error: function(response) { console.log(response) }
    });
}
</code></pre>

<h3>🔎 What This Means:</h3>
<ul>
  <li><code>verifyInviteCode()</code> → Used to check if a code is valid (not useful yet).</li>
  <li><code>makeInviteCode()</code> → Calls an API to tell us how to generate the invite code. <strong>This is our entry point. 🚪</strong></li>
</ul>

<h3>📡 Triggering the Endpoint</h3>

<p>I manually called the <code>/how/to/generate</code> API using <code>curl</code> and formatted the response with <code>jq</code>:</p>

<pre><code>curl -X POST http://2million.htb/api/v1/invite/how/to/generate | jq</code></pre>

<h3>🧩 Output:</h3>

<img width="949" height="230" alt="Screenshot_18" src="https://github.com/user-attachments/assets/166806a5-cd51-4fcc-bb74-56675fef6400" />


<p>This looked like gibberish at first, but experience tells me this is a simple ROT13 cipher. 🔐</p>

<h3>🔄 Decoding with ROT13 (via <a href="https://gchq.github.io/CyberChef/" target="_blank">CyberChef</a>)</h3>

<pre><code>
In order to generate the invite code, make a POST request to /api/v1/invite/generate
</code></pre>

<h3>📬 Generating the Invite Code</h3>

<p>Following the instructions, I made the POST request:</p>

<pre><code>curl -X POST http://2million.htb/api/v1/invite/generate</code></pre>

<p>The response I got looked like this (Base64-encoded):</p>

<img width="743" height="220" alt="image" src="https://github.com/user-attachments/assets/7cf77396-222b-437a-80c0-d787b38d1879" />


<h3>🧪 Base64 Decoding</h3>

<p>I decoded the invite code using CyberChef or this command:</p>

<pre><code>echo "UlE3OUItWERSSVQtR0QzTk8tS1AzMk8=" | base64 -d</code></pre>

<h3>🎉 Invite Code:</h3>

<pre><code>RQ79B-XDIIT-GD3NO-KP32O</code></pre>

<p>✅ Now that we have a valid invite code, we can proceed to register an account and log into the web application. The real fun begins here. 🔓</p>


<br><br><br>
<h2>✅ Step 6: Submitting the Invite Code & Registering an Account</h2>

<p>With the valid invite code obtained from the previous step, I returned to the main web interface at <a href="http://2million.htb/invite" target="_blank">http://2million.htb/invite</a> and entered the code:</p>

<pre><code>RQ79B-XDIIT-GD3NO-KP32O</code></pre>

<h3>⚠️ Important Note:</h3>
<blockquote>
    <strong>Do NOT reload the page</strong> after receiving the invite code. Each time the page loads, a new invite code is generated — and the previous one becomes invalid.
</blockquote>

<h3>📬 Account Registration</h3>
<p>After entering the invite code, I was redirected to a <strong>registration form</strong> where I created a new account using a valid email and password of my choice.</p>

<ul>
  <li><strong>Email:</strong> anything@yourdomain.com</li>
  <li><strong>Password:</strong> anything you prefer</li>
</ul>

<p>After successful registration, I was prompted to log in.</p>

<h3>🔐 Login & Access Granted</h3>

<p>Once I logged in, I was redirected to the main dashboard — a custom-styled interface that mimics the real <a href="https://hackthebox.com" target="_blank">Hack The Box</a> environment. 🎛️</p>
<img width="1365" height="531" alt="image" src="https://github.com/user-attachments/assets/c1cf1c57-b516-45ca-9f06-72152e920a92" />


<p>➡️ With account access now in place, the next step is to explore what functionality is available to us — and how we can exploit it. 🧑‍💻</p>

