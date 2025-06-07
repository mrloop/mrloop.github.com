---
layout: post
title: Scoped CSS in Ruby
categories: [ruby, rails, css, viewcomponent]
excerpt: "The Component-Based CSS Challenge in Rails. After years of working with JavaScript frameworks like Ember and React, I recently returned to Ruby on"
---

## The Component-Based CSS Challenge in Rails

After years of working with JavaScript frameworks like Ember and React, I recently returned to Ruby on Rails for full-stack development. Modern JavaScript frameworks have spoiled us with elegant component-based CSS solutions, but what about Rails?

When I started building UI components with the [ViewComponent](https://github.com/viewcomponent/view_component) library, I was faced a familiar challenge: **how do I scope CSS to individual components without leaks or conflicts?**

While the official ViewComponent docs [previously offered some guidance](https://github.com/ViewComponent/view_component/blob/eba4c1424e58ceadc51d6f948c6ec8eb1b5a982e/docs/guide/javascript_and_css.md) on CSS scoping, these approaches were complex and have since been removed. I also found Julik Tarkhanov's interesting post on [Template Scoped CSS in Rails](https://blog.julik.nl/2025/04/template-scoped-css-in-rails), but it only scopes CSS to top-level templates - not granular enough for component-based development.

I needed something that would:
1. Scope styles to individual ViewComponents
2. Prevent CSS class name collisions
3. Allow component reuse without duplicating styles
4. Maintain the simplicity and developer experience Rails is known for

In this post, I'll walk through the solution I developed and eventually extracted into a gem called [scoped_css](https://github.com/mrloop/scoped_css).

## What We're Building: Component-Scoped CSS in Rails

Ideally, we want to write component templates with encapsulated styles like this:

```html
<style scoped-to-element-below>
  .section {
    border: none;
    scroll-snap-align: center;
    color: purple;
  }
  .heading {
    font-size: 2rem;
  }
</style>

<section class="section">
  <h2 class="heading">Section Title</h2>
</section>
```

And when the component is rendered multiple times, we want the CSS to be included just once:

```html
<style scoped-to-elements-below>
  .section {
    border: none;
    scroll-snap-align: center;
    color: purple;
  }
  .heading {
    font-size: 2rem;
  }
</style>

<section class="section">
  <h2 class="heading">Section Title</h2>
</section>

<section class="section">
  <h2 class="heading">Another Section</h2>
</section>
```

But for this to work properly in Rails, we need a mechanism to transform this into uniquely prefixed CSS selectors.

## The `scoped_css` Helper: How It Works

Here's the helper function that makes component-scoped CSS possible:

```ruby
def scoped_css(&css_block)
  @per_template_outputs ||= Hash.new

  # Capture the CSS content from the block
  css_block_content = ""
  if block_given?
    css_block_content = capture(&css_block)
  end

  # Generate a unique prefix based on the CSS content
  prefix = "a#{Digest::SHA1.hexdigest(css_block_content)}"[0,8]

  styles = Hash.new
  prefixed_css_block_content = Rails.env.local? ? ' <!-- previously output --> ' : ''

  # Check if we've already processed this CSS block
  if @per_template_outputs.has_key?(prefix)
    # Reuse the existing styles mapping but don't output the CSS again
    styles = @per_template_outputs[prefix]
  else
    # Process the CSS block, prefix all class selectors
    prefixed_css_block_content, styles = prefix_css_classes(css_block_content, prefix)
    # Cache the styles mapping for future renders of this component
    @per_template_outputs[prefix] = styles
  end

  result = prefixed_css_block_content.respond_to?(:html_safe) ? prefixed_css_block_content.html_safe : prefixed_css_block_content
  return [result, styles, prefix]
end
```

Lets walk through what's happening:

1. **Content Capture**: We grab the CSS content from the provided block.

2. **Prefix Generation**: We create a unique prefix by hashing the CSS content with SHA1. Why SHA1? It gives us a consistent, unique identifier for identical CSS blocks, allowing us to detect duplicates.

3. **CSS Caching**: The `@per_template_outputs` hash serves as our CSS cache. We check if we've already processed this CSS block (based on its prefix) to avoid duplicating styles.

4. **Prefixing CSS Classes**: For new CSS blocks, we transform each CSS class by adding our unique prefix (handled in the `prefix_css_classes` method, not shown here).

5. **Result**: We return three things:
   - The prefixed CSS (or empty string if already rendered)
   - A mapping of original class names to prefixed ones
   - The unique prefix for this CSS block

The result is CSS that looks like this:

```html
<style>
  .agkd94j4-section {
    border: none;
    scroll-snap-align: center;
    color: purple;
  }
  .agkd94j4-heading {
    font-size: 2rem;
  }
</style>

<section class="agkd94j4-section">
  <h2 class="agkd94j4-heading">Section Title</h2>
</section>

<section class="agkd94j4-section">
  <h2 class="agkd94j4-heading">Another Section</h2>
</section>
```

Our CSS is effectively scoped to the component while ensuring styles are only included once, no matter how many times we render the component.

## Putting It All Together: Complete Example

Let's see how this works in a real Rails application with ViewComponents:

**In your main template (app/views/home/index.html.erb):**

```erb
<% style_string, styles = scoped_css do %>
<style>
  .heading {
    font-size: 3rem;
  }
</style>
<% end %>

<h1 class="<%= styles[:heading] %>">Title</h1>
<%= render SectionComponent.new do %>
  Section 1
<% end %>

<%= render SectionComponent.new do %>
  Section 2
<% end %>

<%= style_string %>
```

**In your component class (app/components/section_component.rb):**

```ruby
class SectionComponent < ViewComponent::Base
end
```

**In your component template (app/components/section_component.html.erb):**

```erb
<% style_string, styles = helpers.scoped_css do %>
<style>
  .section {
    border: none;
    scroll-snap-align: center;
    color: purple;
  }
  .heading {
    font-size: 2rem;
  }
</style>
<% end %>

<section class="<%= styles[:section] %>">
  <h2 class="<%= styles[:heading] %>">Section</h2>
  <%= content %>
</section>

<%= style_string %>
```

This generates HTML with properly scoped CSS:

```html
<!-- app/views/home/index.html.erb -->

<h1 class="atge5q2e-heading">Title</h1>

<!-- app/components/section_component.html.erb -->

<section class="agkd94j4-section">
  <h2 class="agkd94j4-heading">Section</h2>
  Section 1
</section>

<section class="agkd94j4-section">
  <h2 class="agkd94j4-heading">Section</h2>
  Section 2
</section>

<style>
  .agkd94j4-section {
    border: none;
    scroll-snap-align: center;
    color: purple;
  }
  .agkd94j4-heading {
    font-size: 2rem;
  }
</style>

<!-- app/views/home/index.html.erb -->

<style>
  .atge5q2e-heading {
    font-size: 3rem;
  }
</style>
```

## Advanced Usage: Attribute Splatting

Sometimes you need to apply HTML attributes to a component from the parent template. Let's enhance our solution to handle this common pattern:

**In your main template (app/views/home/index.html.erb):**

```erb
<% style_string, styles = scoped_css do %>
<style>
  .section {
    margin: 10px;
  }
  .heading {
    font-size: 3rem;
  }
</style>
<% end %>

<h1 class="<%= styles[:heading] %>">Title</h1>
<%= render SectionComponent.new(attributes: { id: "important-section", class: styles[:section] }) do %>
  <p>Section 1</p>
<% end %>

<%= style_string %>
```

**In your component class (app/components/section_component.rb):**

```ruby
class SectionComponent < ViewComponent::Base
  def initialize(attributes: {})
    @attributes = attributes
  end
end
```

**In your component template (app/components/section_component.html.erb):**

```erb
<% style_string, styles = helpers.scoped_css do %>
<style>
  .section {
    border: none;
    scroll-snap-align: center;
    color: purple;
  }
  .heading {
    font-size: 2rem;
  }
</style>
<% end %>

<section <%= helpers.splat_attributes(@attributes, styles[:section]) %>>
  <h2 class="<%= styles[:heading] %>">Section</h2>
  <%= content %>
</section>

<%= style_string %>
```

The `splat_attributes` helper combines the component's internal class with any external classes passed from the parent:

```html
<section class="agkd94j4-section atge5q2e-section" id="important-section">
  <h2 class="agkd94j4-heading">Section</h2>
  <p>Section 1</p>
</section>
```

> **Note:** Rather than using something like `<section class="<%= styles[:heading] %>" <%= helpers.splat_attributes(@attributes, styles[:section]) %>>`, we use the `splat_attributes` helper to handle both the component's internal classes and any additional attributes passed from the parent.

## CSS Specificity and Customization

What if you want to customize the appearance of a specific component instance? The CSS cascade works in our favor here:

**In your main template (app/views/home/index.html.erb):**

```erb
<% style_string, styles = scoped_css do %>
<style>
  .section {
    margin: 10px;
    color: darkgreen;  /* This will override the component's purple color */
  }
  .heading {
    font-size: 3rem;
  }
</style>
<% end %>

<h1 class="<%= styles[:heading] %>">Title</h1>
<%= render SectionComponent.new(attributes: { class: styles[:section] }) do %>
  <p>Section 1 (Dark Green)</p>
<% end %>

<%= render SectionComponent.new() do %>
  <p>Section 2 (Purple)</p>
<% end %>

<%= style_string %>
```

Since we render style tags at the bottom of each template, CSS specificity works as expected:

1. The component's internal styles are rendered first
2. The parent template styles are rendered later
3. The last declared selector with the same specificity takes precedence

For the first section with the parent's class applied, the color will be dark green (overriding the component's purple). The second section, without the parent's class, will remain purple.

This scoped CSS solution offers several key benefits:

1. **True Encapsulation**: Component styles don't leak or conflict, even with identical class names
2. **Performance**: CSS is only included once per unique component, not per instance
3. **Maintainability**: Components remain self-contained with their styles
4. **Customization**: The CSS cascade still works as expected for intentional overrides
5. **Simplicity**: No additional build steps or preprocessors required

## Conclusion: Component-Based CSS in Rails

By implementing this scoped CSS solution, we've bridged the gap between modern component-based frontend frameworks and Rails ViewComponents. We get the best of both worlds: Rails' simplicity and productivity with the style encapsulation we've come to expect from JavaScript frameworks.

For a drop-in solution, check out the [scoped_css gem](https://github.com/mrloop/scoped_css) I've extracted from this approach. It includes the helpers demonstrated.
