# Notes on modifications made by Crowd Favorite

Installation and basic behavior remain the same.

This is a modified version of the wp-super-cache plugin to allow caching actions to be deferred to WordPress' init action so that cache keys can be modified as needed to allow user/condition specific cache files to be generated. For example: our working need in making this modification was the need to support different levels of authenticated content. This meant that we needed to analyze the user's content permissions and generate a cache file for that specific condition. Since we had many different levels of permissions to handle we needed to be able to generate a profile of the user to use as the cache key. We needed to have WordPress ready and able to analyze user data to do this.

## Upgrading

The plugin relies on the `WPCACHELATE` constant being defined.
If upgrading do one of two things:

- edit the existing wp-cache-config.php to add `define('WPCACHELATE',true);`; or
- delete the existing wp-cache-config.php so that it can be replaced with a version that contains this constants


## List of modifications by file

All modifications in source files are marked with `SP-HAX` comments to mark change locations.

### wp-cache-config-sample.php

- modified to include the definition of a `WPCACHELATE` constant
    - this serves as our flag for delaying cache until init

### wp-cache.php

- added functionality for attaching the delayed cache load until init

### wp-cache-phase1.php

- wrapped all function definitions in `if(!function_exists)` checks to avoid duplicate declaration
- wrapped procedural code in a check to see if it gets run right away vs. being run later
    - this is the OMGHAX part: we need to come back later and wrap this functionality in a function so that we don't need to include the file multiple times to get the procedural code to run

### wp-cache-phase2.php

- modified `wp_cache_phase2()` to return early if it gets called before late cache has had a chance to run phase1
    - this is CYA and probably is not needed overall
- added `wp_cache_do_cache()` function to allow for a complete bail out from cache-writing functions
    - value is processed through the WordPress filter mechanism
    - added check of this function to `wp_cache_get_ob()` to conditionally return from the function to cancel the cache generation
- added `(l)` to cache generation comment for debug assist that cache was performed late


## Modifying cache keys to create separate cache files per user permissions type

We hooked in to `init` to then hook on to `add_cacheaction` and attached a filter to `wp_cache_get_cookies_values`. Here we do our user evaluation and return a key that matches our user type. This forces the generation/use of specific cache keys for each type of user that we need to process. This greatly multiplies the amount of cache files that are generated but allows us to correctly serve authenticated content to those that have privileges and serve login pages or other messages to those who don't.

We also hooked in to the newly added `wp_cache_do_cache` filter to create absolute no-cache generation conditions for certain users and pages in the website.