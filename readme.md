Custom directus extensions
==========================

This is my set of custom-made extensions for [Directus CMS](https://github.com/directus/directus).

Usage
-----

Go to directory with the desired extension.
```
npm install
npm run build
```
Copy the output that appeared in `dist` directory to your Directus install directory to custom extensions folder - usually `public/extensions/custom/interfaces` - [as described here](https://docs.directus.io/extensions/).

Reload Directus App. There you go. 

See https://github.com/directus/extension-toolkit for some more tips.

Note to self: If an interface dows not appear and some crypric error appears, 
such as `Cannot read props of undefined` or some error sduring build process,
this should help:
- delete .cache
- delete node_modules and package-lock.json
- switch to different version of Node.JS (13.6 works, 9.3 should too, v 12.X is problematic)
- npm install again
- it should work again

Also, when build step doesnt work:
- npm i @directus/extension-toolkit@0.8.0

Extensions
----------

- [Location picker](map-picker-interface) - uses Mapy.cz as map engine, has geocoder, enables direct input of coordinates to text fields or clicking on map
- [Color picker](native-html-color-picker) - a color picker using native `<input type="color">`  
