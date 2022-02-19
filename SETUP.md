# Setup
Documenting my process for using this 11ty starter project to build a custom recipe site. [Eleventy needs Node.js v12 or higher.](https://www.11ty.dev/docs/getting-started/). Here is what I am using in development:

```
$ node -v  
v16.2.0
```
---

## 1. Get app running locally 

Use template to create instance. Build and preview default site.

```
$ npm install
$ npm run dev
```

Build for production. Preview build version locally.

```
npm run build
cd ./_site
python -m http.server       
Serving HTTP on :: port 8000 (http://[::]:8000/) ...
```

---

## 2. Inspect app in browser DevTools

By default, apps must be served from an HTTPS-enabled endpoint in order for Progressive Web App capabilities to be accessed. However browsers like [Microsoft Edge](https://docs.microsoft.com/en-us/microsoft-edge/progressive-web-apps-chromium/how-to/) will allow access from _http://localhost_ endpoints for debugging purposes only.

Let's check this out by opening browser DevTools. On my Microsoft Edge browser, I see two options of interest:
 * Applications tab - explore PWA components and status
 * Lighthouse tab - audit app for performance and PWA

When we open the Applications tab, we can see panels for _Manifest_, _Service Workers_ and _Storage_, but nothing much to see here since this app is not yet PWA enabled.

Let's fire up a Lighthouse audit to get some baseline guidance for improvements. I ran audits for both mobile and desktop options - and captured screenshots for both base performance and PWA readiness. Here's what that looks like:

| Desktop Perf | Desktop PWA |
|:---|:---|
| ![](_media/lighthouse-local-desktop.png)|![](_media/lighthouse-local-desktop-pwa.png) |
| Mobile Perf | Mobile PWA |
| ![](_media/lighthouse-local-mobile.png)| ![](_media/lighthouse-local-mobile-pwa.png) |

We can look at improving performance aspects later - but for now let's focus on PWA. We can immediately see the two main issues:

 * _Installable_: No manifest was found.
 * _PWA Optimized_: Does not register a service worker.

The audit also recommends a manual check against the [PWA Checklist](https://web.dev/pwa-checklist/). Let's keep that in mind for later.

---

## 3. Deploy app to HTTPS-enabled endpoint

Let's deploy what we currently have to an HTTPS-enabled server. Most hosting providers will do this for you by default - I chose to deploy my app to [Azure Static Web Apps](https://white-rock-036691f0f.1.azurestaticapps.net/). The process was quick and painless using the [Azure Static Web Apps Extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurestaticwebapps) for VS Code.

 - Log into Azure (via VS Code)
 - Install extension. You see a Static Web Apps pane in Azure.
 - Navigate to it, select an Azure subscription to use.
 - Click `+` to start workflow to add this project to SWA
 - Authorize access to your GitHub repo.
 - Select `Other` for custom configuration
 - Specify `/` for app source, `_site` for build folder
 - Confirm, wait, check the Azure Portal. _App is running!_

The Static Web Apps helper uses the default _build_ script in your _package.json_ and adds the required _GitHub Actions workflow_ to your repository so the app is rebuilt and deployed automatically on every commit.

We have a running PWA in the cloud! You can use the DevTools to run an audit on the deployed version as well. For completeness, I ran it for just desktop mode and captured the performance and PWA scores.

| Performance | PWA  |
|:--- |:--- |
| ![](_media/lighthouse-swa-desktop.png)|![](_media/lighthouse-swa-desktop-pwa.png)|

This seems to have fixed some of the performance best practices issues - but we still have to fix the PWA. Let's focus on that next!

---

## 4. Making Your App a PWA

We know we have to add a manifest, and configure the service worker. And we can do this manually by:
 * Creating a `manifest.json` file with relevant members and adding the relevant `<link>` in app HTML to show its location.
 * Creating a `sw.js` file for service worker implementation and populating it with the relevant lifecycle and functional event handlers for operation, and placing it in the right location of app structure to suit its scope.
 * Registering the service worker in the app code.

But we can also take advantage of helpful PWA tools that make this easier for us. 

For example, for service workers: [Workbox](https://developers.google.com/web/tools/workbox) is a popular set of libraries that can help you create production-ready service workers for your PWA. And the eleventy static site generator I am using provides an [eleventy-plugin-pwa](https://www.npmjs.com/package/eleventy-plugin-pwa) that uses it to generate the service worker, with guidance on how to update the source code to register it and add a Web App Manifest.

We'll look at Workbox and other tools in our upcoming "Developer Tools" week. For now, I wanted to try something a little different.

---

## 5. Getting an assist from PWABuilder

I'm trying out [PWABuilder](https://www.pwabuilder.com/) - a free auditing tool that evaluates your PWA, providing an audit report with actionable options to help you fix identified issues. Just enter your hosted app URL and click Start.

Here's what my audit report looks like. _Sad Trombone_ - we only scored a 30!

![](_media/pwabuilder-audit.png)

 * The good news? That comes from satisfying the _Security_ requirement by using HTTPS, having a valid SSL certificate, and no mixed content on page. 

 * More good news? The _Manifest Options_ and _Service Worker Options_ tabs can help us generate the required _manifest.json_ and _sw.js_ files we need to satisfy installability and network-independent operations for PWA.

Let's go!

--- 

## 5. Configuring Manifest Options

Let's dive into the Manifest Options tab. Here is what that looks like as I start modifying things.

![](_media/pwabuilder-manifest.png)

Some things that make this useful:
 * It pre-populates some fields, making it easier to do small edits.
 * It provides previews showing how manifest options impacts your app on desktop and mobile.
 * It has helpers to generate icons and screenshots!

After doing some cosmetic cleanup (reflecting app structure in project) and manually selecting some [categories](https://developer.mozilla.org/en-US/docs/Web/Manifest/categories), I ended up with this - which I saved to `manifest.json` in the `src` directory of my 11ty project. I saved a subset of the icons in a `src/assets` folder in that same project.

 ```json
 {
    "lang": "en-us",
    "name": "AJ's Offline Cookbook",
    "short_name": "AJ's Cookbook",
    "description": "An offline-friendly online cookbook perfect for that campfire trip off-the-grid",
    "start_url": "/",
    "background_color": "#ff001b",
    "theme_color": "#e8ca6c",
    "orientation": "any",
    "display": "standalone",
    "scope": "/",
    "dir": "ltr",
    "icons": [
        {
            "src": "/assets/logo192.png",
            "sizes": "192x192",
            "type": "image/png"
        },
        {
            "src": "/assets/logo512.png",
            "sizes": "512x512",
            "type": "image/png"
        }
    ],
    "categories": [
        "food",
        "health",
        "book",
        "entertainment"
    ],
    "screenshots": [],
    "shortcuts": []
}
 ```

 We can now update the _eleventy.js_ config file to request that the _src/manifest.json_ and _src/assets/_ folders be passed through to the build.

 The next step is to make sure the app knows the location of the manifest.json file by adding the following to the index.html. In the 11ty project, update the `src/layouts/base.njk` template to make this happen in the build.

 ```
    <link rel="manifest" href="/manifest.json">
 ```

If you are previewing your site as you build it, _inspect_ the Application tab in DevTools and you should see the manifest now show up, with the relevant properties identified. Note that the JSON must be valid! A quick Lighthouse audit from DevTools shows that it still requires a service worker. Let's commit these changes before we tackle that one.

---
