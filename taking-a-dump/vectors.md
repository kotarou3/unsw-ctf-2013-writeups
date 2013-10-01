Vectors
=======

Background
----------
The goal of this challenge was to find which CVE number was the exploit used by the malware

Tools used
----------
* `strings`
* `grep`
* An internet browser

Process
-------
Knowing from Migration that Internet Explorer was injected with the malware, it was likely a Internet Explorer vulnerability. While browsing through `strings` output used in Tracing, I noticed a lot of HTTP request headers were in the memory, including ones to the malware's domain.

I used `strings -n10 dump.vmem | grep -C5 totallylegit\.bxx\.me` to determine the user agent used to access those sites was `Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1)`, or Internet Explorer 6.0.

Now knowing what browser was used, I visited a [list of known exploits for Internet Explorer ordered by available public exploits](http://www.cvedetails.com/vulnerability-list.php?vendor_id=26&product_id=9900&opec=1&order=4) and filtered it down by only including ones that affected IE6 to three CVE numbers: CVE-2008-4844, CVE-2010-0249 and CVE-2007-4790.

I tried these numbers in order as the flag and the second one, CVE-2010-0249, was marked correct.
