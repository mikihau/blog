---
title: "Publishing My First Chrome Extension"
date: 2019-10-14T20:32:09-07:00
draft: false
tags: ["hack", "chrome"]
aliases:
- /post/publishing-my-first-chrome-extension/
---

Just went through the excitement of publishing my first Chrome Extension, Douban Dousay filter ([Chrome Web Store](https://chrome.google.com/webstore/detail/%E8%B1%86%E7%93%A3%E5%B9%BF%E6%92%AD%E8%BF%87%E6%BB%A4%E5%99%A8-douban-dousay-fil/mmlilcjbhccgadmbfojmjlgaegkpklnk)) ([github](https://github.com/mikihau/dousay-filter-chrome-extension)). The code and feature itself is dead simple (~60 lines of javascript), but I just want to make a point for those who thinks making a Chrome Extension is a huge commitment or something. No, it's actually pretty easy to get started.

What I built was a filter for a social network site. The extension should hide an item from the timeline, if the item includes any of the keywords that a user defines. Let me break it up.

- **Modifying the content of an HTML page**  
The Extension starts with a file named `manifest.json`, which includes configuration/metadata of the app. In this file you can specify some `content_scripts`. These `content_scripts` will be injected to the pages that matches the patterns you specify, so they have access to the content of the HTML file.
```javascript
"content_scripts": [
  {
    "js": [
      "content.js"
    ],
    "matches": [
      "https://www.douban.com/",
      "https://www.douban.com/?p=*"
    ]
  }
],
```
Then in `content.js` I can do the DOM manipulations to hide/show DOM objects. It looks like this:
```javascript
// navigate through the DOM tree
var streamItems = document.querySelectorAll("#statuses > div.stream-items > div");
for (var i = 0; i < streamItems.length; i++) {
  for (var filter of filters) {
    if (streamItems[i].outerHTML.search(new RegExp(filter)) != -1) {
      // hide the item by tweaking its style property
      streamItems[i].style.display = "none";
      break;
    }
  }
}
```

- **Adding a popup menu from the extension icon**  
A popup is a page that pops up when you click the extension icon on the top right corner of Chrome. I used to think you can only pop up a dropdown menu with it, but turns out it can be a full blown HTML file to put in whatever you want.
```javascript
"browser_action": {
  "default_popup": "popup.html"
}
```
For lack of better visual design skills, I put a big text box to collect input keywords from users. Maybe it should have been a fancy itemized list, hmm.

![Demo](/images/chrome-extension-popup-demo.png)

- **Storing states**  
This is the big one. I needed to persist the keywords to some kind of storage, so that the same keywords shows up the next time the user opens up the popup menu, and have the filter code use those keywords to modify the DOM properties. So I found the [`chrome.storage` API](https://developer.chrome.com/apps/storage). It's directly accessible to all scripts in the Extension, and it emits events so I can add event listeners like this:
```javascript
// when chrome.storage is changed, reapply filters on the page
chrome.storage.onChanged.addListener(function (changes, areaName) {
  if (areaName == "sync" && changes.hasOwnProperty('filters')) {
    revealDousayItems(); // back to normal 
    hideDousayItems(changes.filters.newValue); // reapply new filters
  }
});
```
On the topic of event listeners -- I found that Chrome [does not allow](https://developer.chrome.com/extensions/contentSecurityPolicy#JSExecution) inline event handlers (e.g. `<button onClick=...>`) for security reasons. That got me trapped for quite a while. Another tricky part for me is to figure out where the code should go, `popup.js` or `content.js`. What helped was to realize that `Popup.js` runs in the context of the popup "window", and knows nothing about the content of the social site page.

That's it! The publishing steps for Chrome Extensions currently involves [getting a developer account](https://chrome.google.com/webstore/developer/dashboard) for $5, then upload the package with some demo images/videos. It's been huge fun for me to hack up my own Extension, and see it up in the Chrome Store. Hopefully this post will encourage more people to start playing with Chrome Extensions, and create their own awesomeness.
