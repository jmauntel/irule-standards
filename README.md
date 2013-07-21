## Description

This is a list of what I consider F5 LTM iRule development best practices.  If you have any ideas on how to improve this list, create a pull request!

---

### STD01:

If environment-specific content is used in an iRule, an environment-specific variables iRule _must_ also be created and be applied as the first iRule for the VIP.  Then, all references to those objects _must_ be done via the variables.  This makes promition of iRule logic from dev => qa => prd simple because the iRules can be _exactly_ the same.

Example: acmevar-dev-irule  

	when HTTP_REQUEST {
		
		# PURPOSE:	Establish environment specific object variables
		#
		
		# Host header definitions, sorted alphabetically
		
		set my_default_host		"web-dev.acme.com"
		set ecommerce_host		"store-dev.acme.com"
		
		# Class definitions, sorted alphabetically
		
		set storeipaddress-class	"storeipaddress-dev-class"
		set eviluseragents-class	"evilUserAgents-dev-class"
		set internalip-class		"internalip-dev-class"
		
		# Pool definitions, sorted alphabetically
		
		set acmecontent-pool		"acme-dev-pool"
	}

Example: acme-dev-irule  

	when HTTP_REQUEST {
	
		# PURPOSE:	Redirects requests based on host header values, or sets the default host header value
		#
		
		set my_luri [string tolower [HTTP::uri]]
		
		if { not ( [HTTP::host] equals ${my_default_host} ) } {
			switch -glob [HTTP::host] {
				"buy.acme.com" {HTTP::respond 301 "location" "https://${ecommerce_host}" }
				default { HTTP::respond 301 "location" "https://${my_default_host}[HTTP::uri]" }
			}		
		}
	}


* Notice the use of the **${my\_default\_host}** and **${ecommerce\_host}** variables in the acme-dev-irule iRule.

---

### STD02:

Use `HTTP::respond` instead of `HTTP::redirect` for redirects.  The `HTTP::respond` method has more flexability with controlling the response to the client.

Example:  

	if { [HTTP::uri] starts_with "/server-status" } {
		HTTP::respond 301 "location" "/"
	}

---


### STD03:

All iRule logic _must_ written in a way that can be testable via an external automation suite like **irule-tester**.  All iRule logic _should_ be developed using TDD (Test-Driven Development) principles, meaning:

* Tests _must_ be written before the iRule code
* Tests _must_ then be run to validate that the test fails before any changes have been made to the iRule
* iRule code is then written to verify that the test criteria has been met

---

### STD04:

Each test case _must_ be preceded by the following logic header:

* Date it was created/modified
* Date it will expire
* Who requested it
* Description of why it was requested

Example: 

	# 2012-10-24, 2013-10-24, IT Security, Discard all requests matching malicious URLs
	should_discard 0001 http://${targetSite}/evil-index.html

---


### STD05:

Do not not create variables that only duplicate pre-existing ones if they are not necessary. i.e., use built in variables when possible (ex. [HTTP::host] )

---

### STD06:

Make comparison logic as specific as possible. Use "equals" before "contains", "starts_with", etc. The preferred order of precedence is:

* equals
* start_with
* ends_with
* regex
* contains

---

### STD07:

Standardize on "string tolower" or "string toupper".  Use the one you pick, always.  

---

### STD08:

When using switch statements, leverage the "default" case, where appropriate.

Example:

        when HTTP_REQUEST {
        
                # PURPOSE:      Redirects requests based on host header values, or sets the default host header value
                #
                
                set my_luri [string tolower [HTTP::uri]]
                
                if { not ( [HTTP::host] equals ${my_default_host} ) } {
                        switch -glob [HTTP::host] {
                                "buy.acme.com" {HTTP::respond 301 "location" "https://${ecommerce_host}" }
                                default { HTTP::respond 301 "location" "https://${my_default_host}[HTTP::uri]" }
                        }               
                }
        }

---


### STD09:

Decide on a standard order to apply your iRules to your virtual servers.  My preference is as follows:

* when CLIENT_ACCEPTED
  * SNATs (case specific)
* when HTTP_CLASS_SELECTED
  * ASM enable/disable (case specific)
* when HTTP_REQUEST
  * System Protection (block malicious requests)
  * Module & Feature Management (enable/disable modules and features, compression)
  * Sorry Page Logic
  * Host Header Manipulation
  * HTTP Redirects (Simple - switch)
  * HTTP Redirects (Complex - if/elseif/else)
  * Pool Selections
* when HTTP_RESPONSE

---

### STD10:

Use `return` to exit iRule events unless you have a specific reason to use `event disable`.  The `return` exists the event for this request, while `event disable` disables the current event for all connections on the current TCP session.

---

### STD11:

Use `discard` vs `reject`.  The `discard` action silently drops the user request, while using `reject` sends a TCP reset to the requester.  The `discard` action will slow down evil user requests due to the client connection staying open until its connection timeout has expired.

---

### STD12:

When possible, use relative paths when using `HTTP::respond`.  This allows the logic to be protocol agnostic (HTTP vs. HTTPS), which means the same rule can be used for SSL and non-SSL virtual servers, reducing the complexity of maintaining the same iRule logic in SSL and non-SSL iRules.

Example:

	Relative redirect:
		HTTP::respond 301 "location" "/my/new/location.html"

	Absolute redirect:
		HTTP::respond 301 "location" "http://www.acme.com/my/new/location.html"

---

### STD13:

Write redirect and pool selection logic so it can be used on an SSL and non-SSL VIP in the same rule.  Wrap SSL-specific logic in an if or switch block that triggers based on virtual server port.  This reduces logic duplication, and management overhead by allowing you to apply the same iRule to SSL and non-SSL virtual servers.

Example:

	if { [HTTP::uri] starts_with "/server-status" } {
		HTTP::respond 301 "location" "/"
	}
	
	if { [TCP::local_port] == "443" } {
	        if { [HTTP::uri] equals "/ssl/only/request.html" } {
			HTTP::respond 301 "location" "/ssl/only/destination.html"
		}
	}

---

### STD14:

Only create log statements for troubleshooting, and remember to delete them afterward.  Unnecessary log entries in the audit logs can slow down troubleshooting.

---

### STD15:

Add spacing between each if block when using if, elseif, else to increase readability.  Don't be afraid of whitespace, it's free.

Bad Spacing Example:

	if { [HTTP::uri] starts_with "/req-one" } {
		HTTP::respond 301 "location" "/dest-one"
	}
	if { [HTTP::uri] starts_with "/req-two" } {
		HTTP::respond 301 "location" "/dest-two"
	}
	if { [HTTP::uri] starts_with "/req-three" } {
		HTTP::respond 301 "location" "/dest-three"
	}

Good Spacing Example:

	if { [HTTP::uri] starts_with "/req-one" } {
		HTTP::respond 301 "location" "/dest-one"
	}

	if { [HTTP::uri] starts_with "/req-two" } {
		HTTP::respond 301 "location" "/dest-two"
	}

	if { [HTTP::uri] starts_with "/req-three" } {
		HTTP::respond 301 "location" "/dest-three"
	}

---

### STD16:

Include all redirects in a single if block, when possible. This makes multiple matches less likely.  When multiple redirect matches are found for a request, the **first** match wins, but a message is sent to the LTM log that indicates there were multiple matches found.  This can cause unexpected results.

Mutiple-match Example:

	if { [HTTP::uri] starts_with "/req-one" } {
		HTTP::respond 301 "location" "/dest-one"
	}

	if { [HTTP::uri] starts_with "/req-one-two" } {
		HTTP::respond 301 "location" "/dest-two"
	}

	if { [HTTP::uri] starts_with "/req-one-two-three" } {
		HTTP::respond 301 "location" "/dest-three"
	}

Single-match Example: (although still a bad design)

	if { [HTTP::uri] starts_with "/req-one" } {
		HTTP::respond 301 "location" "/dest-one"
	} elseif { [HTTP::uri] starts_with "/req-one-two" } {
		HTTP::respond 301 "location" "/dest-two"
	} elseif { [HTTP::uri] starts_with "/req-one-two-three" } {
		HTTP::respond 301 "location" "/dest-three"
	}

---

### STD17:

Include all pool selections in a single if block, when possible. This makes multiple matches less likely.  When multiple pool selections are made for a request, the **last** selection wins.  This can cause unexpected results.

Mutiple-match Example:

	if { [HTTP::uri] starts_with "/req-one" } {
		pool acmeone-dev-pool
	}

	if { [HTTP::uri] starts_with "/req-one-two" } {
		pool acmetwo-dev-pool
	}

	if { [HTTP::uri] starts_with "/req-one-two-three" } {
		pool acmethree-dev-pool
	}

Single-match Example: (although still a bad design)

	if { [HTTP::uri] starts_with "/req-one" } {
		pool acmeone-dev-pool
	} elseif { [HTTP::uri] starts_with "/req-one-two" } {
		pool acmetwo-dev-pool
	} elseif { [HTTP::uri] starts_with "/req-one-two-three" } {
		pool acmethree-dev-pool
	}

---

### STD18:

Decide on an object naming standard and use it, always.

Example:

* names are all lowercase
* ${name}-${env}-${object_type}
  * Examples: acme-dev-irule, internaladdresses-qa-class

---

### STD19:

Create a class if the same iRule logic is repeated two or more times for similar content.

Duplicate Example:

	if { [HTTP::uri] equals "/req-one" } {
		pool acme-dev-pool
	}

	if { [HTTP::uri] equals "/req-two" } {
		pool acme-dev-pool
	}

	if { [HTTP::uri] equals "/req-three" } {
		pool acme-dev-pool
	}

Class Example:

	if { [class match [HTTP::uri] equals acme-dev-class} ] } {
		pool acme-dev-pool
	}

---

### STD20:

Perform a code review for all iRule changes.  Use the buddy system!

