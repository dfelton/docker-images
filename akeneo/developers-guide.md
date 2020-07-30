
# Akeneo Developers Guide

### Table of Contents 

- Preface
- Hello World
- Auto-detected Bundle YAML files
- Routing
- Form extensions
  - extensions.json
  - Overriding / re-using existing form extensions

--- 
### Preface

An effort to lay out development practice guidelines, and unravel the mysteries of the universe.

Akeneo is an application built on top of the [Oro platform](https://oroinc.com/oroplatform/). 
The Oro platform is an open source PHP platform built on symfony, intended to provide a 
framework for building web applications. One thing to note however is that Akeneo has decided to
fork Oro, and have added to their `composer.json` `replace` configurations so that Oro is loaded
directly from `akeneo/pim-community-dev` itself rather than from Oro themselves (a summary of what
they have changed from Oro's native code is not available here).

Being that Oro is built on Symfony, development in Akeneo also highly follows Symfony practices.
One of the first things to note, is that despite being on Symfony 4.4, at the time of writing Akeneo
does not utilize Symfony Flex. While Akeneo themselves have included it in the `composer.json` of 
`akeneo/pim-community-dev`, developers working on a project derived from `akeneo/pim-community-standard`
are not to have `symfony/flex` added to the root of the project. Hence, feature customizations to 
the root project (our repo) are still accomplished by following the organization structure of Symfony's
[Bundle system](https://symfony.com/doc/4.4/bundles.html).

Frontend development on Akeneo, is a bit of a mixed bag of tools. At a high level:

- Templating is accomplished with [Twig](https://twig.symfony.com/) 
- [requireJS](https://requirejs.org/) and [backbone.js](https://backbonejs.org/) are heavily used for rendering just 
  about anything and everything which is displayed to the user
- [Friends of Symfony Js Routing](https://symfony.com/doc/master/bundles/FOSJsRoutingBundle/usage.html) is used to 
  expose PHP routes to frontend JavaScript
- [Oro Datagrids](https://doc.oroinc.com/backend/entities/data-grids/) are a common tool for displaying paginated rows 
  of information to the user
- [Oro Translations](https://doc.oroinc.com/backend/translations/translations/) handles internationalization
- [less](http://lesscss.org/) is Akeneo's CSS pre-processor of choice
- [webpack](https://webpack.js.org/) is used to manage processing the frontend assets of the application's various 
  bundles, mapping out dependencies and outputting the compiled results into a distribution folder for serving assets 
  to the client browser.
- [Form extensions](https://docs.akeneo.com/2.3/design_pim/overview.html#create-our-first-form-extension) is a concept 
  created by Akeneo to manage building out UI components for the frontend in a tree-like structure through the use of 
  YAML configurations. Form extensions are associated to a requireJS module which handles rendering of the extension
  based on the logic of the requireJS module and the configuration of extension itself, and (optionally) a template file.
  Whereas Magento 1 has layout XML and PHP "Block" classes, Akeneo has Form extensions and requireJS modules. 


--- 

### Hello World

As mentioned earlier, feature development and application customization at its core is accomplished through the use of 
Symfony's Bundle system. A bundle, in its most simple form, can potentially consist of strictly one file (it's "Bundle" 
file), and one additional line of PHP to register said bundle.

Bundles are to be organized inside a vendor directory (name of the vendor that wrote it, not composer's vendor 
directory) within `src/`, inside a `Bundle` directory of said vendor, and then the bundle's directory itself, which 
will always be suffixed with "Bundle". 

Example: `src/{VendorName}/Bundle/{BundleName}Bundle/`

For our example we'll propose a `HelloWorld` bundle, of the `Grommet` vendor.

File: `src/Grommet/Bundle/HelloWorldBundle/GrommetHelloWorldBundle.php`
 
```php
<?php

namespace Grommet\Bundle\HelloWorld;

use Symfony\Component\HttpKernel\Bundle\Bundle; 

final class GrommetHelloWorldBundle extends Bundle
{

}
```

Then, register it by adding a line in our project's `config/bundles.php` file:

```php
return [
    \Grommet\Bundle\HelloWorldBundle\GrommetHelloWorldBundle::class => ['all' => true],
];
```

That's it! We have our first bundle. It does absolutely nothing, but it is a valid, registered bundle to our application.
Now, lets discuss a few things here:

- In our `confing/bundles.php` file when registering our bundle, noticed we've chosen to add: `['all' => true]`. This 
  tells our application to go ahead and load this bundle during runtime for all environments. If we wanted to, we 
  could choose to have our bundle only loaded for specific environments, such as `dev` or `prod` (referencing the 
  `APP_ENV` variable of our projects `.env` file)
- While it may seem overly verbose, it is good practice to prefix the vendor name to the bundle name, when 
  defining the class name of the bundle 
  
  (`class GrommetHelloWorldBundle extends Bundle`)  
 
  Doing so will always help ensure we avoid namespace collisions with any other bundles we may introduce into the 
  application.
  
**Important:** The name given to the bundle class, will directly translate into the string(s) used for mapping within 
various areas of our application. The map will always exclude the word "Bundle", will sometimes this will be all lowercase, 
sometimes it will map exactly, and sometimes map using underscores between studly cased words of the class name.

Examples of this mapping occurring (some of this below will make more sense later when delving into individual topics
more deeply, for now we're stressing the importance of how a bundle's name yields specific automatic mapping across the 
application):

- Public assets for our bundle above will be compiled to: `public/bundles/grommethelloworld/`. During localhost 
  development, these are symlinked, for upper environment they are created as hard copies. Regardless, this directory
  will hold the assets a bundle's `Resources/public/`directory (full path for our bundle: `src/Grommet/Bundle/HelloWorldBundle/Resources/public/`)

- When editing a module's `requirejs.yml` file, path mappings will follow the same "all lowercase, no underscore" 
  convention as the bullet point above. This is because we are providing mapping values relative from the root of 
  `public/bundles/` directory.
  ```yml
  config:
    paths:
      # RequireJS module example:
      grommet_hello_world/foo: grommethelloworld/js/foo
  
      # Template Example:
      grommet_hello_world/template/coolpage/column/left: grommethelloworld/templates/coolpage/column/left.html
  ```
  The left side (key) of these mappings could technically be whatever we want. They just have to be unique across the 
  application. However, it is good practice to prefix them with the [Symfony alias of your bundle name](https://symfony.com/doc/4.1/bundles/best_practices.html#bundle-name).
  
  The right side (value) of these mappings **must** be prefixed with the lower case bundle name, followed by the path
  to the file it is providing a map to relative from our bundle's `Resources/public` directory. Per the convention of 
  requireJS, mappings for javascript files may exclude `.js` as requireJS handles this for us. 
  
  So, in summary, with the configuration above, we would expect our Bundle to contain these two files:
  ```
  src/Grommet/Bundle/HelloWorldBundle/Resources/public/js/foo.js
  src/Grommet/Bundle/HelloWorldBundle/Resources/public/templates/coolpage/column/left.html
  ```    
  After compiling our application's frontend assets, these two files will be exposed to the web here:
  ```
  public/bundles/grommethelloworld/js/foo.js
  public/bundles/grommethelloworld/templates/templates/coolpage/column/left.html
  ``` 
  (note that an HTTP request's URL path would exclude `/public` since requests are served from the root of 
  this directory already).

- An example of exact mapping with the bundle name, is when creating a [Symfony Bundle Extension](https://symfony.com/doc/4.1/bundles/extension.html#creating-an-extension-class).
  In this situation, Symfony will automatically look for a `DependencyInjection` folder inside our bundle, 
  and immediately within that directory a class which follows the naming convention `{BundleName}Extension`, 
  dropping the suffix `Bundle` and instead adding `Extension`. So for our bundle, symfony would look for 
  `src/Grommet/Bundle/HelloWorldBundle/DependencyInjection/GrommetHelloWorldExtension.php`

- [Symfony recommends](https://symfony.com/doc/4.1/bundles/best_practices.html#routing) (technically they state "must", 
  but doesn't appear to be enforced in any way) that a bundle's route names be prefixed with the alias of the bundle 
  (`grommet_hello_world` in our case). Per [Symfony's documentation](https://symfony.com/doc/4.1/bundles/best_practices.html#services)
  this same rule applies to the naming of any services a bundle provides. 

In summary, the name of your bundle is important in numerous ways, and expect to have to reference it in a few
different formats depending on what it is you are trying to accomplish at the time. 

That's a wrap for our Hello World segment on creating our first bundle, and the significance in choosing a name for our 
bundle.

---

### Auto-detected Bundle YAML files

Akeneo Bundles may have many YAML files inside the bundle's `Resources/` directory. A few of these, follow either 
Symfony, Oro, or Akeneo conventions and will be automatically detected if present, and added to our application's 
configuration (still need to recompile, but no specific registration of its existence is required). A summary of these
files are:

- DataGrid Definitions: `config/datagrid/*.yml` (subdirectories within `datagrid` allowed)
- Form Extension Definitions:
    - `config/form_extensions.yml`
    - `config/form_extensions/*.yml` (subdirectories within `form_extensions` allowed)
- ACL Rules: `config/acl.yml`
- ACL Groups: `config/acl_groups.yml`
- RequireJS paths and config: `config/requirejs.yml`
- Translation Files (replacing `{LOCALE_CODE}` with an actual value, such as `en_US`):
    - `translations/jsmessages.{LOCALE_CODE}.js`
    - `translations/messages.{LOCALE_CODE}.js`

Depending on the size of your bundle, you may have a desire to further organize your configurations into more specific
files named accordingly to what they do. In this situation, we must manually register these files via a 
[Symfony Bundle Extension](https://symfony.com/doc/4.1/bundles/extension.html#creating-an-extension-class). As discussed
in the Hello World section above, this file will reside exactly at `DependencyInjection/{BundleName}Extension.php`. It 
must extend the class `Symfony\Component\HttpKernel\DependencyInjection\Extension`. Finally, within the class' `load()`
method we can include our additional YAML files for our application:

In this example below, we add three more yaml files to our application, which do not follow built-in naming conventions, 
but help organize our ever-growing code base:

```php
<?php

namespace Grommet\Bundle\HelloWorldBundle\DependencyInjection;

use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;
use Symfony\Component\HttpKernel\DependencyInjection\Extension;

/**
 * Class GrommetHelloWorldExtension
 * @package Grommet\Bundle\MakerBundle\DependencyInjection
 */
final class GrommetHelloWorldExtension extends Extension
{
    public function load(array $configs, ContainerBuilder $container)
    {
        $loader = new YamlFileLoader($container, new FileLocator(__DIR__ . '/../Resources/config'));
        $loader->load('services.yml');
        $loader->load('datagrid_actions.yml');
        $loader->load('datagrid_listeners.yml');
    }
}
``` 

---

### Routing

If your bundle provides new routes, or modifications to existing routes, this will require registering your bundle's 
routes. This is done in `config/routes/routes.yml`. Technically, any YAML file in this directory will automatically be
picked up by the application's Kernel, so we could just add all of our routes there. However, in order to maintain 
organization we'll simply append to the existing `routes.yml` the registration of additional 
[routing resources](https://symfony.com/doc/4.1/routing/external_resources.html) for each bundle. 

```yaml
# config/routing/routes.yml

# notice route name matches bundle's alias (mainly to ensure uniqueness, not strictly enforced)
grommet_hello_world:
    # notice @GrommetHelloWorldBundle exactly matches name of bundle class (strictly enforced) 
    resource: "@GrommetHelloWorldBundle/Resources/config/routing.yml" 
```

After registering to our application the external routing resource, we can now define routes within our bundle itself:

```yaml
# src/Grommet/Bundle/HelloWorldBundle/Resources/config/routing.yml

# name here must be unique, Symfony best practice is to prefix with alias of Bundle 
grommet_hello_world_index:
    # the URL path for our route
    path: /grommet/hello-world/
```

Once registering the route itself, we must now define a requireJS controller for it:
```yaml
# src/Grommet/Bundle/HelloWorldBundle/Resources/config/requirejs.yml
config:
    paths:
        # The key here just needs to be unique, but a naming convention would be: {bundle_alias}/controller/{controller-name}
        # The value here is the path to hello-world.js, from `public/bundles`, therefore notice the all lowercase bundle mapping convention in the value
        grommet_hello_world/controller/hello-world: grommethelloworld/js/controller/hello-world
    config:
        pim/controller-registry:
            controllers:
                # "grommet_hello_world_index" exactly matches the name of the route we defined in the bundle's "routing.yml"
                grommet_hello_world_index:
                    # value of "module" matches exactly the path we setup a few lines above, in this same file
                    module: grommet_hello_world/controller/hello-world
                    # value matches an existing ACL resource defined in any bundle's "Resources/config/acl.yml" file 
                    aclResourceId: name_of_acl_resource

```

And lastly (/s), define the requireJS controller itself:

```js
// src/Grommet/Bundle/HelloWorldBundle/Resources/public/js/controller/hello-world.js
'use strict';
define(
    [
        'pim/controller/front',
        'pim/form-builder'
    ],
    function (BaseController, FormBuilder) {
        return BaseController.extend({
            renderForm: function (route) {
                return FormBuilder.build('name-of-a-form-extension-to-render').then((form) => {
                    form.setElement(this.$el).render();
                });
            }
        });
    }
);
```

In this example, `name-of-a-form-extension-to-render` would be replaced with the actual value of a form extension we've
defined. Form extensions are covered in the next segment.

Some things to note on the above, is what we've all changed, and what recompiling is going to be necessary to see these
changes reflected. So far we've:
- Added / Modified Route(s), therefore `bin/console fos:js-routing:dump --target=/var/www/m2/pim/public/js/routes.js`
  will need to be executed to expose these PHP routes to JavaScript.
- We've modified our bundle's `Resources/config/requirejs.yml` file, therefore `bin/console pim:installer:dump-require-paths`
  will need to be executed to regenerate `public/js/`

---

### Form extensions

If we want to add our own content to a page, we're almost certain to need a form extension. The overwhelming majority
of form extensions from out of the box Akeneo, come from the `PimUIBundle` 
(`vendor/akeneo/pim-community-dev/src/Akeneo/Platform/Bundle/UIBundle/`). The `PimUIBundle` provides much of the 
groundwork for the frontend of the application, and Akeneo's various bundles provide feature specific enhancements
to this baseline. For this reason, the `PimUIBundle`'s `Resources/config/form_extensions` directory is always a good 
place to look when wondering "how did they build that".

- Form extensions are YAML configurations that are then translated into JSON.
  - At its minimum, each form extension will have a corresponding requireJS module
  - [See here](https://docs.akeneo.com/2.3/design_pim/overview.html#what-does-it-mean) for available configuration 
    options for form extensions, and what they mean.
- requireJS controllers handle frontend routes. These controllers utilize the `pim/form-builder` requireJS module to 
  render a form extension to the client.
- For a walk through on creating a new form extension, please [see Akeneo's docs](https://docs.akeneo.com/2.3/design_pim/overview.html#). 
  They'll walk you through outputting "Hello World" to the screen, whereas here we'll focus on the subtle nuances
  of working with form extensions not exactly covered in the documentation.

  
#### extensions.json

As mentioned, form extensions are translated into JSON (output to `public/js/extensions.json`). Because of this, any YAML changes to our extension requires 
that this JSON be recompiled for our requireJS modules. There are a few ways to do this:

```shell script
# execute the node script directly
docker exec tg-m2-php sh -c 'cd /var/www/m2/pim; NODE_PATH=node_modules node vendor/akeneo/pim-community-dev/frontend/build/update-extensions.js'

# execute akeneo's webpack, updating of form extensions is included as a preBuild script
docker exec tg-m2-php yarn run --cwd /var/www/m2/pim webpack
# webpack with watch mode (will detect changes and recompile extensions.json on save)
docker exec tg-m2-php yarn run --cwd /var/www/m2/pim webpack --watch

# execute webpack with our configuration (strips out less compiling and a few other tools)
docker exec tg-m2-php sh -c 'cd /var/www/m2/pim; NODE_PATH=node_modules node node_modules/webpack/bin/webpack.js --config /var/www/m2/pim/dev/webpack/js-and-form-extensions.js'
# our webpack again, with watch mode
docker exec tg-m2-php sh -c 'cd /var/www/m2/pim; NODE_PATH=node_modules node node_modules/webpack/bin/webpack.js --config /var/www/m2/pim/dev/webpack/js-and-form-extensions.js --watch' 
```

There are going to be pros and cons to every way chosen. If you're strictly changing the contents of the YAML for a form
extension, you may be able to get away with just directly executing `update-extensions.js` with node and see your change
reflected on page refresh, depending on the type of change. However, this is often not the case, as when developing in Akeneo you will most likely find 
yourself working in numerous files at once (configuring the form extension itself, modifying the requireJS module for the 
form extension, modifying the requireJS controller that will render the form extension on that route, 
tweaking the routes that the registered the requireJS controller, the list goes on). Because of this, we'll be 
an entire section of this guide to localhost development workflow.

#### Overriding / re-using existing form extensions

##### Overriding 

Overriding configurations for form extensions of other bundles is easy. Simply reference the named key of the form 
extension you want to configure differently, and begin configuring it. Take the `pim-product-create-modal` as an example
(found in `vendor/akeneo/pim-community-dev/src/Akeneo/Platform/Bundkle/UIBundle/Resources/config/form_extensions/product/create.yml`): 
```yaml
extensions:
    pim-product-create-modal:
       module: pim/form/common/creation/modal
       config:
           labels:
               title: pim_common.create
               subTitle: pim_enrich.entity.product.plural_label
           picture: illustrations/Product.svg
           successMessage: pim_enrich.entity.product.flash.create.success
           editRoute: pim_enrich_product_edit
           postUrl: pim_enrich_product_rest_create
```

If we wanted this form extension to utilize our own requireJS module, instead of `pim/form/common/creation/modal`, we'd 
simply add form extension configuration to our bundle like so:

```yaml
extensions:
    pim-product-create-modal:
        module: grommet_hello_world/form/create-product-modal
```

(This example assumes we've already revised our module's `requirejs.yml` file to include path mappings for this module).
As you can see, it is simply a matter of defining a YAML definition with matching keys leading up to the value you'd 
like to override.

##### Re-using

Re-using existing form extensions on the other hand, is not quite as simple, if the goal is to re-use it elsewhere (as 
in, on another page than originally written for) with modifications that apply just to the new spot for it. The reason
for this is form extensions are built out like a tree. Each branch / leaf has but one parent. Each branch / leaf 
exists as its own definition. So while we can certainly register a new route, setup a new controller module, and have
our new controller render out the form extension desired to be re-used, this won't yield the desired outcome. This is
because once we start making customizations to the re-used form extension, we are in fact customizing both occurrences
of it.

On the plus side, we can simply re-define a new form extension with essentially all the same configurations (and start
editing _that_ definition, independent of the original). This for more complex structures, this causes a fair amount
of copy / paste of YAML, but achieves the desired outcome.
 
Going back to the `pim-product-create-modal` example above, if we wanted to re-use on our own page, while modifying 
a few components to our needs (for only our page), we simply setup a new definition and modify what we need:

```yaml
extensions:
    grommet-product-create-modal:
       module: grommet/form/common/creation/modal
       config:
           labels:
               title: "So cool, I don't need i18n" 
               subTitle: "Subtitle waaaat"
           picture: illustrations/Product.svg
           successMessage: pim_enrich.entity.product.flash.create.success
           editRoute: pim_enrich_product_edit
           postUrl: pim_enrich_product_rest_create
```

Now we can register a requireJS module for our new route, and have that module render out this form extension of ours.
If other form extensions had the original as a target parent, they too, will need to be duplicated out to build out 
the intended rendering outcome expected (which is why I say this can lead to a lot of copy / paste).

