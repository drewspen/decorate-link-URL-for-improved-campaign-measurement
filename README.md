# decorate-link-URL-for-improved-campaign-measurement
GTM decorate link URL for improved campaign measurement

Previously I created a repository regarding writing URL query string parameters to first party cookies. This solution builds on that by passing the UTM campaign values to your CRM. As before, the inspiration for this was an Analytics Mania blog post from years ago. 

This example is for either a Pardot or HubSpot iframe CRM form lead generation implementation. Unlike the earlier 'Write URL query strings 2 cookies' solution, this is not a Sandboxed JavaScript GTM template due to the limitations of Sandboxed JavaScript.

First in the GTM container for the 'parent' page (regardless of the iframe form appearing later in the site customer journey), we use the earlier 'Write URL query strings 2 cookies' solution to persist the incoming campaign information (as a setup tag on the GA4 initialization).

Next we 'listen' for the iframe possessing the desired CRM host (Pardot, HubSpot, etc). Typically this is a DOM ready trigger, but for an SPA it would be on history change 'pushState'. In the JavaScript, the hosts are a list of possibilities, as are the elements containing URLs on the page. Alter and narrow these as needed.  The alterations are made to 'Parent target URL link hostname found on page', 'Parent decorate URL link with UTM names and values from cookies' and 'Parent decorate URL link with _ga cookie name and value'.  

Once found, the page elements have the UTM and _ga client id appended as URL query string parameters to the element URLs. Why is this necessary? In the case of an iframe, the browser treats the iframe as a completely separate object. This solution is a relatively simple way for the CRM imbedded in the iframe to 'know' about the parent page anonymous browser client id and UTM campaign information.

Now, in the GTM container inside the iframe, 'Write URL query strings 2 cookies' persists the campaign information to cookies as a setup tag on the GA4 initialization. On the GTM Configuration Setting variable, the iframe URL is checked for the parent anonymous browser client id. If absent, the client id is set to he value sourced the Analytics Storage variable type. 

From here sourcing the campaign information into hidden fields on the CRM forms can vary greatly based on your implementation. If your 'utm_campaign=my_campaign_name' was passed from your parent site URL, to your parent site cookie, to your iframe URL, to your iframe cookie, to a like named hidden field on your iframe CRM form, that code MIGHT resemble the following. And the triggering will likely be based on a visibility trigger for the form page element.

<script>
function getCookie(name) {
// JavaScript code for reading cookies goes here...
}
document.querySelector('input[data-sc-field-name="utm_campaign"]').value = getCookie('utm_campaign');
</script>

The entire solution is available from my GitHub as a downloadable JSON that can be imported into a GTM container. 

My solution does not address getting the conversion metric from the CRM form submit back to the 'parent' page. For that I recommend Simo Ahava's work.

How Do I Use The postMessage() Method With Cross-site Iframes?
https://www.teamsimmer.com/2023/05/02/how-do-i-use-the-postmessage-method-with-cross-site-iframes/

Finally, here is Julius Fedorovicius' work that inspired mine. 

Transfer UTM Parameters From One Page To Another with GTM
https://www.analyticsmania.com/other-posts/transfer-utms-from-one-page-to-another-with-gtm-version-1/
https://www.analyticsmania.com/post/transfer-utm-parameters-google-tag-manager/
