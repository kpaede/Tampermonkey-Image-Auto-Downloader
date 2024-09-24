# Tampermonkey-Image-Auto-Downloader
This Tampermonkey script automatically downloads all JPG images from a Website as you browse, using the original filenames.

## Installation
Install Tampermonkey Extension

Download and install the Tampermonkey extension for your browser. Create a New Script.

Click on the Tampermonkey icon in your browser toolbar.
Select "Create a new script..." from the dropdown menu.
Add the Script Code

Remove any placeholder code in the editor.

Paste the following script code into the editor:

## Browser Configuration
To ensure images are downloaded automatically without prompts, adjust your browser settings:

Chrome
Turn Off "Ask where to save each file before downloading"

Navigate to chrome://settings/downloads.
Toggle off "Ask where to save each file before downloading".
Allow Automatic Downloads

Go to chrome://settings/content/automaticDownloads.
Under "Customized behaviors", click "Add" next to "Allowed to automatically download multiple files".
Enter Website and click "Add".

## Legal Disclaimer
Ensure that downloading images complies with the website's terms of service and applicable laws.


```// ==UserScript==
// @name         Auto-Download JPGs Including Viewer Content
// @namespace    http://tampermonkey.net/
// @version      2.2
// @description  Automatically downloads all JPG images from the current site as you browse, including images loaded in viewers and iframes, using original filenames.
// @author
// @match        *://*/*
// @grant        GM_download
// @grant        GM_xmlhttpRequest
// @connect      *
// ==/UserScript==

(function() {
    'use strict';

    const downloadedImages = new Set();

    // Intercept network requests
    (function(open) {
        XMLHttpRequest.prototype.open = function(method, url) {
            this.addEventListener('load', function() {
                handleNetworkRequest(this.responseURL);
            });
            open.apply(this, arguments);
        };
    })(XMLHttpRequest.prototype.open);

    (function(fetch) {
        window.fetch = function() {
            const promise = fetch.apply(this, arguments);
            promise.then(response => {
                if (response.url) {
                    handleNetworkRequest(response.url);
                }
                return response;
            });
            return promise;
        };
    })(window.fetch);

    function handleNetworkRequest(url) {
        // Only proceed if the URL points to a JPG image
        if (!url.match(/\.(jpe?g)$/i)) {
            return;
        }

        // Normalize URL
        let normalizedUrl;
        try {
            normalizedUrl = new URL(url, window.location.href).href;
        } catch (e) {
            console.error('Invalid URL:', url);
            return;
        }

        // Prevent duplicates
        if (downloadedImages.has(normalizedUrl)) {
            return;
        }
        downloadedImages.add(normalizedUrl);

        // Extract filename
        const pathname = new URL(normalizedUrl).pathname;
        let filename = pathname.substring(pathname.lastIndexOf('/') + 1);
        if (!filename) {
            console.warn('Could not extract filename from URL:', normalizedUrl);
            return;
        }

        // Download the image
        GM_download({
            url: normalizedUrl,
            name: filename,
            saveAs: false,
            onerror: function(err) {
                console.error('Error downloading image:', err, 'URL:', normalizedUrl);
            }
        });
    }

    // Process images in the main document and iframes
    function processDocument(doc) {
        processImagesInDocument(doc);
        observeDocument(doc);

        // Process iframes within the document
        const iframes = doc.querySelectorAll('iframe');
        iframes.forEach(iframe => {
            processIframe(iframe);
        });
    }

    function processImagesInDocument(doc) {
        const images = doc.querySelectorAll('img');
        images.forEach(img => {
            processImageElement(img);
        });

        // Process background images
        processBackgroundImages(doc);
    }

    function processImageElement(img) {
        // Handle 'src' attribute
        if (img.src) {
            downloadImage(img.src);
        }
        // Handle 'srcset' attribute
        if (img.srcset) {
            const srcsetUrls = img.srcset.split(',').map(s => s.trim().split(' ')[0]);
            srcsetUrls.forEach(downloadImage);
        }
        // Handle lazy-loaded images
        if (img.dataset && img.dataset.src) {
            downloadImage(img.dataset.src);
        }
        if (img.dataset && img.dataset.srcset) {
            const dataSrcsetUrls = img.dataset.srcset.split(',').map(s => s.trim().split(' ')[0]);
            dataSrcsetUrls.forEach(downloadImage);
        }
    }

    function processBackgroundImages(doc) {
        const elements = doc.querySelectorAll('*');
        elements.forEach(el => {
            const style = window.getComputedStyle(el);
            const backgroundImage = style.getPropertyValue('background-image');
            if (backgroundImage && backgroundImage !== 'none') {
                const urlMatches = backgroundImage.match(/url\(['"]?(.*?)['"]?\)/);
                if (urlMatches && urlMatches[1]) {
                    const bgUrl = urlMatches[1];
                    downloadImage(bgUrl);
                }
            }
        });
    }

    function processIframe(iframe) {
        try {
            const iframeDocument = iframe.contentDocument || iframe.contentWindow.document;
            if (iframeDocument) {
                processDocument(iframeDocument);
            }
        } catch (e) {
            // Cross-origin iframe
            console.warn('Cannot access iframe content due to cross-origin restrictions.');
        }
    }

    function observeDocument(doc) {
        const observer = new MutationObserver(mutations => {
            mutations.forEach(mutation => {
                if (mutation.type === 'childList') {
                    mutation.addedNodes.forEach(node => {
                        if (node.nodeType === Node.ELEMENT_NODE) {
                            if (node.tagName === 'IMG') {
                                processImageElement(node);
                            } else if (node.querySelectorAll) {
                                const imgs = node.querySelectorAll('img');
                                imgs.forEach(processImageElement);

                                // Process background images in new nodes
                                processBackgroundImages(node);

                                // Process iframes in new nodes
                                const iframes = node.querySelectorAll('iframe');
                                iframes.forEach(processIframe);
                            }
                        }
                    });
                } else if (mutation.type === 'attributes') {
                    if (mutation.target.tagName === 'IMG') {
                        processImageElement(mutation.target);
                    } else if (mutation.attributeName === 'style') {
                        processBackgroundImages(mutation.target.ownerDocument);
                    }
                }
            });
        });

        observer.observe(doc.documentElement, {
            childList: true,
            subtree: true,
            attributes: true,
            attributeFilter: ['src', 'srcset', 'data-src', 'data-srcset', 'style']
        });
    }

    function downloadImage(url) {
        // Normalize the URL
        let normalizedUrl;
        try {
            normalizedUrl = new URL(url, window.location.href).href;
        } catch (e) {
            console.error('Invalid URL:', url);
            return;
        }

        // Check if the URL points to a JPG image
        if (!normalizedUrl.match(/\.(jpe?g)$/i)) {
            return;
        }

        // Prevent duplicates
        if (downloadedImages.has(normalizedUrl)) {
            return;
        }
        downloadedImages.add(normalizedUrl);

        // Extract filename
        const pathname = new URL(normalizedUrl).pathname;
        let filename = pathname.substring(pathname.lastIndexOf('/') + 1);
        if (!filename) {
            console.warn('Could not extract filename from URL:', normalizedUrl);
            return;
        }

        // Download the image
        GM_download({
            url: normalizedUrl,
            name: filename,
            saveAs: false,
            onerror: function(err) {
                console.error('Error downloading image:', err, 'URL:', normalizedUrl);
            }
        });
    }

    // Start processing the main document
    processDocument(document);

    // Intercept network requests
    // (Already included at the top of the script)

})();
```
