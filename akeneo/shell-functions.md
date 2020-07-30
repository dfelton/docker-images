
This will help someone who has been banging their head against the wall wondering why their changes do not show:

```
pimBuildRoot() {
  START=`date +%s`
  echo Stopping nginx docker container
  docker container stop tg-m2-nginx > /dev/null
  echo Blowing away all cache and generated files
  docker exec $PIM_BUILD_CONTAINER find m2/pim/public/bundles -type l -delete
  docker exec $PIM_BUILD_CONTAINER rm -rf \
      m2/pim/var/cache \
      m2/pim/public/bundles \
      m2/pim/public/js \
      m2/pim/public/css \
      m2/pim/public/dist \
      m2/pim/public/cache
  docker exec $PIM_BUILD_CONTAINER mkdir m2/pim/public/js
  echo Dumping Symfony Routes to JavaScript
  docker exec $PIM_BUILD_CONTAINER m2/pim/bin/console fos:js-routing:dump --target=/var/www/m2/pim/public/js/routes.js
  echo Setting up symlinks to bundle public directories
  docker exec $PIM_BUILD_CONTAINER m2/pim/bin/console assets:install --relative --symlink
  docker exec $PIM_BUILD_CONTAINER ln -rsT /var/www/m2/pim/vendor/akeneo/pim-community-dev/src/Akeneo/Connectivity/Connection/front/src /var/www/m2/pim/public/bundles/akeneoconnectivityconnection-react
  echo Dumping the paths for all the requirejs.yml files for each bundle
  docker exec $PIM_BUILD_CONTAINER m2/pim/bin/console pim:installer:dump-require-paths
  echo Dumping oro js-translations
  docker exec $PIM_BUILD_CONTAINER m2/pim/bin/console oro:translation:dump en_US
  echo Compiling LESS
  docker exec $PIM_BUILD_CONTAINER sh -c 'cd /var/www/m2/pim; NODE_PATH=node_modules node vendor/akeneo/pim-community-dev/frontend/build/compile-less.js'
  echo Updating form extensions
  docker exec $PIM_BUILD_CONTAINER sh -c 'cd /var/www/m2/pim; NODE_PATH=node_modules node vendor/akeneo/pim-community-dev/frontend/build/update-extensions.js'
  echo compiling application javascript
  docker exec $PIM_BUILD_CONTAINER sh -c 'cd /var/www/m2/pim; NODE_PATH=node_modules node node_modules/webpack/bin/webpack.js --config /var/www/m2/pim/dev/webpack/js-only.js'

  echo Starting nginx docker container
  docker container start tg-m2-nginx > /dev/null
  END=`date +%s`
  echo Pim rebuild completed in $((END-START)) seconds
}
pimBuild() {
    PIM_BUILD_CONTAINER="tg-m2-php"
    pimBuildRoot
}
pimBuildDebug() {
    PIM_BUILD_CONTAINER="tg-m2-php-xdebug"
    pimBuildRoot
}
```
