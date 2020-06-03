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

Extensions
----------

- [Location picker](map-picker-interface) - uses Mapy.cz as map engine, has geocoder, enables direct input of coordinates to text fields or clicking on map
- [Color picker](native-html-color-picker) - a color picker using native `<input type="color">`  
