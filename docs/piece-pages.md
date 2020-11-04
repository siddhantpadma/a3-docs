---
title: "Piece pages"
---

# Piece pages in A3

:::  tip Note:
**"What's a piece page?"**
<!-- I realize that for alpha 1 our audience is more familiar with Apostrophe, but this is a topic that comes up even among people who know a lot about Apostrophe, so I'd like this note to stay in. Let's work on making it as clear as possible. Tom -->

The most intuitive example is a blog: if the individual [piece](pieces.md) is one blog post, then the piece page is the blog's home page, where all of the posts can be discovered, paginated, filtered and explored. In addition,  "piece pages" are responsible for serving the individual webpage for each piece.

If the project has a piece module called `product`, then that module provides a way to create, edit, manage and query pieces. If the project has a piece page module called `product-page`, then that module provides a way to browse and view the pieces.

In general piece modules are concerned with *editing and APIs*, while piece page modules are concerned with *browsing as part of a website.* For developers familiar with the "model / view / controller" pattern: **piece modules are the model layer for your content type, while piece page modules are the view layer.**

Some sites just need a widget module for each piece, but most will want to let users view a full-blown webpage for each one, or at least browse and paginate through a complete list. And that's where piece pages shine.
:::

## Piece pages by example

In A3, piece pages work much like 2.x, with a few important changes. Most of those changes are related to the [new module format](module-format-example.md), which you should definitely read about if you haven't already.

Just like in A2, piece page modules and piece modules come in pairs. And the name of the piece page module automatically determines which piece module it works together with. So we don't need very much code in our actual module in order to get started.

Here's an example of a piece page module that works with the product piece described in the [pieces section](pieces.md):

```js
// in ./app.js
require('apostrophe')({
  modules: {
    // piece module
    product: {},
    // piece page module
    'product-page': {}
  }
});
```

```js
// in ./modules/page/index.js, add the new page type
// to the "Type" dropdown for new pages

module.exports = {
  options: {
    types: [
      {
        name: 'default-page',
        label: 'Default'
      },
      {
        name: 'product-page',
        label: 'Product Index'
      },
      {
        name: '@apostrophecms/home-page',
        label: 'Home'
      }
    ]
  }
};
```

```js
// In ./modules/product-page/index.js, configure
// our new piece page type
module.exports = {
  extend: '@apostrophecms/piece-page-type',
  options: {
    label: 'Product Index Page'
  }
};
```

```markup
{# in ./modules/product-page/views/index.html #}

{% import '@apostrophecms/pager:macros.html' as pager with context %}
{% extends "layout.html" %}

{% block main %}
  {% for product in data.pieces %}
    <h4><a href="{{ product._url }}">{{ product.title }}: {{ product.price }}</a></h4>
    <section>{% area product, 'description' %}</section>
  {% endfor %}
  {{ pager.render({ page: data.currentPage, total: data.totalPages }, data.url) }}
{% endblock %}
```

```markup
{# in ./modules/product-page/views/show.html #}
{% extends "layout.html" %}
{% set product = data.piece %}

{% block main %}
  {# The layout already output the title for us #}
  <h4>Price: {{ product.price }}</h4>
  <section>{% area product, 'description' %}</section>
 % endblock %}
```

That's all we need to create a basic paginated index page for all of our products, with "virtual" subpages for the individual products, based on the `slug` field of each piece.

### Major changes from A2

* In A3, we extend `@apostrophecms/piece-page-type`.

* In A3, there are no tags, so there is no "with these tags" feature to limit what is displayed on a particular piece page. However, you can do this yourself as described later.

## Filtering pieces on the piece page

Just like in A2, we can configure `piecesFilters` to offer filtering to our website visitors. Let's start by adding a field to our `product` pieces that's good to filter on:

```js
// In ./modules/product/index.js
module.exports = {
  // ...
  fields: {
    add: {
      // ... add this as one more field
      color: {
        type: 'select',
        label: 'Color',
        choices: [
          {
            value: 'red',
            label: 'Red'
          },
          {
            value: 'green',
            label: 'Green'
          },
          {
            value: 'blue',
            label: 'Blue'
          }
        ]
      }
    }
  }
};
```

```js
// In ./modules/product-page/index.js
module.exports = {
  extend: '@apostrophecms/piece-page-type',
  options: {
    label: 'Product Index Page',
    piecesFilters: [
      {
        name: 'color',
        label: 'Color'
      }
    ]
  }
};
```

```markup
{# in ./modules/product-page/views/index.html #}

{# ... add this before the list of products #}
<nav>
  {% for choice in data.piecesFilters.color %}
    <a
      class="{{ 'current' if data.query.color == choice.value }}"
      href="{{ data.url | build({ color: choice.value }) }}"
    >{{ choice.label }}
    </a>
  {% endfor %}
</nav>
```

:::  tip Notes:
You won't see any choices for the filter unless you actually have products that have been assigned a color via the color field we just added. Similarly, if you add more than one filter, you will never see filter combinations that produce zero results.

The syntax for `piecesFilters` may change before the final 3.x release.
:::

## Multiple piece pages for the same piece type

Rather than filtering on the page, you might prefer to separate page as a "home" for red products, green products, and blue products... or split them up in some other way. It's up to you. The important thing is that you let Apostrophe know how to identify the pieces you want for this particular page. This is different from A2, where this was handled via tags by default.

We'll solve it by adding a `color` field to our piece pages as well, along with logic to browse only matching pieces and assign the right URL to each piece:

```js
// In ./modules/product-page/index.js
module.exports = {
  extend: '@apostrophecms/piece-page-type',
  options: {
    label: 'Product Index Page'
  },
  fields: {
    add: {
      color: {
        type: 'select',
        label: 'Color',
        choices: [
          {
            value: 'red',
            label: 'Red'
          },
          {
            value: 'green',
            label: 'Green'
          },
          {
            value: 'blue',
            label: 'Blue'
          }
        ]
      }
    }
  },
  methods(self, options) {
    return {
      filterByIndexPage(query, page) {
        if (page.color) {
          query.color(page.color);
        }
      },
      chooseParentPage(pages, piece) {
        return pages.find(page => page.color === piece.color);
      }
    };
  }
};
```

Here we've done three things:

* We've added a `color` field to the piece page itself.
* We've overridden the `filterByIndexPage` method in order to restrict the `product` pieces to those that match the `color` of this piece page. Notice that we didn't have to write a query builder for `query.color`. All `select` fields automatically have one.
* We've overridden `chooseParentPage` to pick the first piece page with a `color` setting that matches the `piece`. This helps Apostrophe assign the right `_url` to the piece.

## Infinite scroll and refresh-free filtering

These A2 features don't currently exist in A3. Since A3 is much less opinionated on the front end, they probably won't be part of the core, but they may come back at some point as an optional module.