---
layout: post
title: How not to be depressed when working with WordPress
subtitle: A 2019 sane vision for a cleaner WordPress architecture
categories:
- blog
catalog: true
date:       2019-04-08
author:     Tristan Bessoussa
header-img: /images.unsplash.com/photo-1519110437047-c6488cf2051d?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1567&q=80
tags:
    - WordPress
    - Architecture
    - composer
    - best practices
---

Ok, so you chose to start a new WordPress project (or someone chose for you) ? Well good news, it's 2019, it doesn't have to suck from the start anymore; Here's what I advise you to follow.

In this article, I'm not going to explain everything in details, that would require me to write a book about it. I will just point out directions: What you can do and how to do it. It's designed for everyone, web developers, backend developers, front-end developers...

I promise you, *no more than 3 minutes of reading* per category.

In the company I work for, we have 150+ WordPress sites. I made the decision for the new ones to adopt everything I wrote in this article, and slowly migrate the legacy one. Already 15% of them are using this architecture at the time I was writing this article.

Note: I'm accepting [pull requests](https://github.com/tristanbes/devops-life) to fix english mistakes if you catch them.

---

# Use Bedrock Edition

It's **THE** starting point of this article. Bedrock is a WordPress boilerplate where you can install WordPress as a package via Composer.

The boilerplate, created by [Roots](https://roots.io/), leverages the use of modern development tools and embrace the [12 Factor app methodology](https://12factor.net/).

The main advantages of this boilerplate are:

## Support of environment variables

If you worked with modern software applications, you're familiar with the concept of having all of your application configuration stored as environment variables.

It allows you to ease having multiple environments such as `production`, `staging` and `development` where you can for example only enable sending emails on `production`, without the need of constantly changing values depending on which environment you are.

In 2019, every major frameworks has a concept of environnement variables in the PHP world: Symfony, Laravel, Yii. It's a common practice that exists since decades on Windows and Unix world.

Environement variables can be stored as a file named `.env.example` which you have to commit on your git repository. They look a lot like this:

```bash
DATABASE_URL=mysql://john_doe:p4sSWorD@127.0.0.1:3306/db_awesome_blog
WP_ENV=development
WP_HOME=http://my-awesome-wordpress.vm
WP_SITEURL=${WP_HOME}/wp
```

## Better directory structure

Everything is organized better and as a result, it's easier to find what you were looking for. The structure look likes:

```bash
├── composer.json
├── config
│   ├── application.php
│   └── environments
│       ├── development.php
│       ├── staging.php
│       └── production.php
├── vendor  <-- where the 3rd party code lives
└── web
    ├── app
    │   ├── mu-plugins
    │   ├── plugins
    │   ├── themes
    │   └── uploads
    ├── wp-config.php
    ├── index.php
    └── wp <-- WordPress files
```

Bonus point to security since the web root (entry point of your application) `web/index.php` is isolated from the rest of the structure.


## Reproductible builds

The concept of [reproductible builds](https://en.wikipedia.org/wiki/Reproducible_builds) can sum up as: no matter when or where i'm going to build and deploy the application, I will always have the same version of the code (and plugins)

Since you're going to be working on you local environment (if it's not already the case 👀), you need to guarantee that what you have on your computer will be exactly the same that what you'll have on your production. Let me take an example on why this is important.

Let's say you've been running a website in production for like a year until there's a nasty hardware crash on you server. Now all your files are lost. Luckily for you you did things right and had your `uploads/` directory and database backup somewhere in the cloud.

The next step will be to re-install WordPress using the database and media backup. Then re-install the 20 plugins you had on that website. In one year, plugins evolve (and it's good), but your theme, or some of your features might not be compliant with the version of 3.5 of WooCommerce. It might result of a broken website

Another example is when you're working as a team on a project, without reproductable builds you will end up with the frontend person working with WordPress 5.1.1 while the developper still has version 4.9.10 and won't be able to reproduce the bug you have with that version.

Because of this concept, you'll need to **forbid any type of plugins/themes installation or updating from the WordPress backend**. By default Bedrock will take care of this when you are in the `production` environment. Since WordPress is also a composer dependency, updating it is as simple as `composer update roots/wordpress`

But how to guarantee that ? Well the answer is just below: use **Composer**.

# Use Composer

Bedrock boilerplate allows you to manage dependencies of your project using [Composer](https://getcomposer.org/) like **any** modern and decent PHP project should do (WordPress, I'm looking at you right now).

Do you want to install `WooCommerce` ? Go inside your terminal and type:

`composer require woocommerce/woocommerce`

It'll fetch the latest version and freeze the version you got on your `composer.lock`. This file is what guarante you to get always the same version of everything no matter where you download your project dependencies, or where you do it. Because of the importance of this file, it has to be commited ⚠️.

All the plugins are also installable via composer; How ? **3 scenarios are possible**:

- The plugin provides a `composer.json`; The developper or company behind the plugin is not locked in a WordPress thing of the past syndrome and has developed modern applications. This would mean that their plugin also ships a [composer.json][https://github.com/woocommerce/woocommerce/blob/master/composer.json] file that allow installation through composer. With this scenario you can search for your plugin inside [packagist.org](https://packagist.org/?query=woocommerce) which is a repository containing all the public PHP packages in the universe ✨.


- The plugin does not provide a `composer.json`; If you don't see the plugin on [packagist.org](https://packagist.org/), then the plugin does not have a `composer.json` file.
Don't worry, **wpackagist** comes to the rescue. Outlandish created this project to offer a way to install 100% of the plugins and themes using the main WordPress repository. So, this particular WordPress plugin cannot be found on packagist.org, so let's use [wpackagist](https://wpackagist.org/) instead:

`composer require wpackagist-plugin/cookie-law-info`

- The plugin is a paid/private one and does not have a `composer.json`. You'll have multiple options from there:
   - [Open a ticket to their support]((https://github.com/elementor/elementor/issues/4042)) or on their Github and ask to provide a way to install their plugin via composer. You are their customer, you are paying for a product that you can't install in a proper way. They should care. Bad players are Advanced Custom Field, Elementor and good player is [Deliciours brains](https://deliciousbrains.com/wp-migrate-db-pro/doc/installing-via-composer/) (Migrate DB Pro, Offload Media...).
   - [Use alternative solutions](https://github.com/PhilippBaschke/acf-pro-installer) when available.
   - Create a new private Github repository (it's free), copy/paste the code there and add a [composer.json like in this example](https://gist.github.com/tristanbes/fbacfb2ce6990e7fdc7411a73715fd92), tag a new release using [Github releases](https://help.github.com/en/articles/creating-releases), and then require this package using a custom repository. All the procedure is [listed in detail on roots.io website](https://roots.io/guides/private-or-commercial-wordpress-plugins-as-composer-dependencies/). I'd advise, if you have the budget for, to use [Private Packagist](https://packagist.com/) to host your private packages.

A good article can be found on [how using composer with WordPress](https://roots.io/using-composer-with-wordpress/) on roots.io website.

# Use a better templating system than PHP: "I'm yelling timber" !

What about the templating system ?
PHP is not a good templating system. Hard to read, hard to write, and let's not fool ourselves, we're no longer in the early 2010, we have tools created especially for each use case. The best PHP templating system is [Twig](https://twig.symfony.com/). It is widely use in the Symfony ecosystem (but not only).

So how can we use Twig inside a WordPress project ? Luckily for us, there's a project called  [Timber](https://timber.github.io).

> Plugin to write WordPress themes with object-oriented code and the Twig Template Engine

It adds a View / Controller model that will have a huge benefit on the readability and the maintenance of your theme.

Once you start writing your themes using Twig, **you won't go back to plain PHP**. Period.

**Pro tip**: Since you're not designing your themes with plain PHP anymore but with Twig, the server needs to transform your `.twig` files into `.php` files. This phase is called compilation. It adds extra time to do that. Luckily there's a cache system that stores the compiled templates on your file system so Twig doesn't recompile your twig templates on each request that your server needs to process.

By default, Timber does not activate that cache. Here's how to activate it safely you'll save up to 40% of generation time by doing so.

{% include image.html width="688" url="/img/timber-cache.png" description="Difference of rendering time before/after enabling Twig cache system. <a href='https://blackfire.io/profiles/compare/5b79d23e-3bb8-4089-8e56-82aaae679b6b/graph'>More details on this Blackfire trace</a>." %}

Inside your theme's `functions.php`

```php
// Activate Twig caching.
if (class_exists('Timber') && WP_ENV === 'production') {
    Timber::$cache = true;
}

// Replace WP_ENV === 'production' by !WP_DEBUG if you are NOT in a Bedrock structure
```

Note that it caches the the template and not the data. Any edits made to the `.twig` templates on the production won't be reflected unless you clear the cache. (Who edits files on production though ? 😱 Certainly not you !)

# Prefer storing your assets on the cloud

Storing your assets on the local filesystem under `uploads/` directory is nice, but at some point, you'll have to synchronize this folder through all your environements, otherwise your local version will a complete different set of medias than the production and vice-versa.

Remember when we talked about reproducable builds ? Wouldn't it be nice if we can store our media remotly so we don't use local filesystem ? Spoiler alert, yes, it's nice.

The one other major advantage of this technique is **allow your website to scale** to recieve a huge pike of traffic load. All you have to do is to add more servers (horizontal scaling), since you have a reproductible build, the project and plugins will be the same on server number 1, 2, ... 23, 24.

Services you can use to store your media on the cloud :
- [Amazon S3](https://aws.amazon.com/s3/). The most famous, yet, not the cheapest one 💸.
- [Digital Ocean Spaces](https://www.digitalocean.com/products/spaces/). S3-compatible, 45 locations available.
- [Scaleway](https://www.scaleway.com/object-storage/). Cheap storage, S3-compatible but only 2 locations available (Paris and Amsterdam).

✅ Remember to always use a bucket location near where the majority of your audience live.
I would advise you to choose a "S3 compliant API" since there’s not much WordPress plugin that allow you to use a remote storage that work in majority of use-case.

Here's a list of free plugins that allow you to store your media on different cloud storage:

- [MediaCloud](https://wordpress.org/plugins/ilab-media-tools/) (free)
- [humanmade/s3-uploads](https://github.com/humanmade/S3-Uploads) (free)

My wish for WordPress is for them to use an filesystem abstraction such as [Flysystem](https://flysystem.thephpleague.com/docs/). This way, we no longer have to rely on plugins to hack into WordPress to tell it to upload medias and assets elsewhere. Until this happens, you can use one of the plugins above.


# Know tools you're working with: ACF can hurt your website performance

If you're a user of Advanced Custom Field, known as ACF, then you must know some of the performance bottleneck you might have if you're abusing the `get_field` function.

You must only expose only the variables you need per page. I've seen a lot of example, even in the documentation where you are invited to do

```php
$context['options'] = get_fields('options'); // retrieve all options fields from the database

Timber::render('index.twig', $context); // render the index.twig page with the options passed to the twig context so you can use them inside your view.
```

When you do that, it will trigger one database call per field you're getting. So let's say you have 35 fields under the option category, it'll trigger 35 database call just to get those fields, but, do do you really need those 35 fields on all pages ?

What to do instead ?

```php
$context['options']['twitter_link'] = get_fields('twitter_link', 'options');
$context['options']['logo_footer_img'] = get_fields('logo_img', 'options');
```

```twig
{# use them inside your .twig template; #}
<footer>
    <a href='{{ options.twitter_link }}'>Follow us</a>
    <img src='{{ options.logo_footer_img }}' />
</footer>
```

And what about videos and images fields ? If you are using them, you should know that if you retrieve them "as is" using the `get_field('my_youtube_video')`, the code will try to detect the metadata (size, orientation, lenght etc..) of the videos and images.

What's the matter here ? Well, since you're hosting your medias on the cloud (or a remote service, eg: YouTube in this example), the video and images has to be downloaded on the server side to allow retrieve this information you won't need in 99% of the case. In this situation doing external HTTP calls from your server to the internet adds precious extra time and kills your webperformance.

I've seen project spending almost 1second doing external calls, preventing the server to send HTML to the browser by fetching just 10 images information from the cloud.

In those cases you need to use `get_field('my_youtube_video', null, false)`. The last option (`false`), tells to retrieve the field (in this case, the YouTube url of the video, without any formatting or aditionnal treatment). One other case is to set the configuration of the field as a "URL" field, (and not oEmbed), and for images, "Image URL"

Explicit is better than implicit. It helps your colleague to provide a more accurate code review of your work. That work is harder when you don't know what your template has access to.

In real world project, doing this kind of optimization leads to important performance improvements.

{% include image.html width="688" url="/img/blackfire-acf.png" description="Difference of rendering time before/after retrieving only the strict necessary ACF fields. -850 requests to the database and -37% of rendering time ! <a href='https://blackfire.io/profiles/compare/3984573f-3fc2-48fc-9683-2668c3a4a7e5/graph'>More details on this Blackfire trace</a>." %}


# Lint your code

Is your theme shipping CSS, Javascript and some PHP ? You should lint them, make sure you write code compliant with industry standard. Use:

* [ESlint](https://eslint.org/) - Javascript
* [stylelint](https://stylelint.io/) - CSS
* [Prettier](https://prettier.io/) - Javascript, CSS...
* [PHP-CS-Fixer](https://cs.symfony.com/) - PHP (Example of our [.php_cs_dist file that contains linting rules](https://gist.github.com/tristanbes/8f29b6f9336a77fd9b205f3453aae892))

Use whatever building tool/task runner you feel confortable with ([Gulp](https://gulpjs.com/), [Brunch](brunch.io)). If you don't have picked up a tool yet, and you need a simple one, I'd suggest you take a look at **[yProx-CLI](https://github.com/Yproximite/yProx-cli)**

For your own sanity, please don't open code you're downloading from the plugins you use. You might be shocked by the poor code quality you're dealing with and might end up re-writing a whole lot of WordPress plugins.

# Use a continuous integration (CI) service

Use a CI (Travis, Gitlab, CircleCI, Jenkins)
Here's our base `.travis.yml` file that ensure our `composer.json` is valid, that our PHP Code is linted and same goes for our JavaScript files; No pull requests should be merged if you have linting errors that will never be fixed once they hit the production.

```
dist: trusty

language: php
php:
  - '7.3'

cache:
  yarn: true
  directories:
    - ./vendor
    - ./node_modules
    - ~/.composer/cache/files

before_install:
  - |
    if [ -f "package.json" ]; then
        set -e # properly fails Travis if one of the following commands fails
        nvm install v10
        nvm use 10
        node --version
        npm --version
        npm install --global yarn
        set +e # re-enable previous behaviour
    fi

script:
  - composer validate
  - composer install
  - vendor/bin/php-cs-fixer fix -v --dry-run --using-cache=no
  - if [ -f "package.json" ]; then yarn install; fi;
  - if [ -f "package.json" ]; then yarn lint; fi;
  - if [ -f "package.json" ]; then yarn build; fi;
  ```

# Setup an automated dependency updates tool

Since we have 150 WordPress projects in our organization, running `composer update` on all of them once per week is

We were using [ManageWP](https://managewp.com/) to update WordPress and all of their dependencies, but this option is now off the table since it breaks the concept of reproductible builds. Nothing should alter/mutate your environement. The code should be exactly the same like in your git repository.

One viable option for us was to delegate the updating process to [Dependabot](https://dependabot.com/) which job is to open a pull request when updates are available. Alternatives exists, for exemple, [Renovate](https://renovatebot.com/).

We configured it to automatically merge the pull requests if the package update is a major or a minor (`v1.0.0 -> v1.1.4` will be merged automatically, but `v1.0.0 -> v2.0.0` won't);

⚠️ The WordPress ecosystem does not necessarly respect the [SemVer](https://semver.org/), because... (I'm still looking for a reason...).

Since we choose a "Platform as a Service" (PaaS) to host all of our WordPress, as soon as a PullRequest gets merged on master branch, the project gets re-deployed on the production.

No matter which hosting you chose, you should have automatic deployment, otherwise, the project will be up-to-date but will never hit the production if you need to re-deploy each project if you have a huge ammount of them.

# Choose the right hosting

I would strongly advise to go for a PaaS. The main advantages of a Platform as a Service (PaaS) are:
- No maintenance needed for the server or the software
- Automated deployments
- Vertical, horizontal scaling ready
- Cheap
- Easy
- Isolated (no cross contamination from one site A to site B when hosted on the same server)

Some example of good PaaS are:
- [🌍 Heroku](https://www.heroku.com/) (from 14$/month)
- [🇫🇷 Scalingo](https://scalingo.com/) (from 10,80€/month)
- [🇫🇷 Clever Cloud](https://www.clever-cloud.com/en/) (from 26,50€/month)

⛔️ Don't ever use a FTP or a SFTP to deploy to the production. Since you need to build your project by running `composer install` and maybe build assets by running `yarn build`, the long dead process of copy/paste the project on your FTP are **definitly and finally** dead ⛔️

# Note for WordPress plugins developpers

In order to make the WordPress ecosystem a better place please, you have to respect at least 5 things:

⚠️ Don’t asssume that your plugin user will stores their uploaded files on the local file system. **Assume they don’t**. Abstract your file manipulation by using for instance [Flysystem](https://flysystem.thephpleague.com/docs/).

The main reason behind this reasoning is:
- When hosting your application on the cloud, you might want to be able to scale your application by adding new servers.
- When you host your WordPress on an [immutable infrastructure](https://www.digitalocean.com/community/tutorials/what-is-immutable-infrastructure), which basically means
    - Either you’re not able to write to the server, preventing any modifications
    - Either the modifications your made are lost when the project gets redeployed. (A deploy happens each time you merge code on your code repository or when you scale (add or remove servers) on your application. Only what is on your git repository gets deployed on the server.

⚠️ Provide a way to configure your plugin through environment variables, and use a fallback using the database if the environement variable is not found.
`Environment variables > Database`; This allow us to automate plugins installation and put all the configuration in one place.

⚠️ Provide a way to install your plugin with **composer**, it requires a `composer.json` file with 10 lines of code at the minimum. Just watch some examples [here](https://github.com/deliciousbrains/wp-amazon-s3-and-cloudfront/blob/master/composer.json), [here](https://github.com/awesomemotive/WP-Mail-SMTP/blob/master/composer.json#L4) or [here](https://github.com/Yoast/wordpress-seo/blob/trunk/composer.json). The next step is to tag your releases using [Github Releases](https://github.com/deliciousbrains/wp-migrate-db/releases) respecting SemVer and submit your package to [packagist.org](https://packagist.org/packages/submit) by entering the Github URL of your repository.

⚠️ Support [maintained version of PHP](https://www.php.net/supported-versions.php)

⚠️ Lint your PHP code using PHP-CS-Fixer FFS ! Choose between PSR2 or Symfony preset, but use one and fix your code.