CacheWagon -- Making your Rails caching-experience more fun!
============================================================

cache_wagon is a library of utilities that will greatly enhance your caching experience when working with Ruby on Rails.

Requirements
------------

* Rails 3.x (Not tested with Rails 3.1)
* Jquery
* The [Babilu](https://github.com/toretore/babilu) gem

Installation
------------

In your gemfile:

    gem 'cache_wagon'

Then run the installation-generator, which will add the required javascript-files to your public/javascripts folder

    rails generate cache_wagon:install

cache_wagon automatically gets included in the :defaults javascript expansion, so you should have the following in your layout:

    <%= javascript_include_tag :defaults %>

If you instead wish to include the javascript-files manually, you will this in your layout instead:

    <%= javascript_include_tag 'locales','date','cache_wagon' %>

Note the extra 'locales' reference. This is a reference to the locales.js file that the babilu gem automatically generates.
You can leave this reference out, if you're already using the babilu gem and have the locales.js referenced elsewhere.

License
-------

This work is licensed under a Creative Commons Attribution-ShareAlike 3.0 Unported License

Usage
=====

Currently cache_wagon has the following utilities:

* cacheable_time_ago_in_words
* cacheable flash messages
* cacheable_dynamic_block

Each will be described in the following sections.

cacheable_time_ago_in_words
===========================

Have you ever tried caching pages where this helper is used? It's no fun at all!

Imagine you have a page, where you print out a list of comments, and on each comment you print out a time_ago_in_words string for when the comment was created.
Then imagine that you page_cache this page indefinitely and sweep it when a comment is added, removed or edited.
What happens then with the time_ago_in_word strings? - They only change when the page is sweeped!
So you risk having a page where a comment is marked as created "less than a minute ago" even though it might be years since that comment was created.

What you can do instead, is use cache_wagon's cacheable_time_ago_in_words method, which instead of outputting a time_ago_in_words-string simply outputs the date localized with the :short format.
Furthermore the date is put in a span-tag with some HTML5-data attributes that stores the UTC-version of the date.
Then when the page loads, an unobtrusive piece of javascript-code finds all these span-tags and replaces their content with a similar time_ago_in_words string based on their respective UTC-date and the local time on the clients computer.
The javascript that generates the time_ago_in_words string is a direct port of the original rails-version, and even uses your rails locale-files to generate the strings (with help from the babilu gem).

**Example:**

    cacheable_time_ago_in_words(Time.zone.now - 2 minutes) # => <span class="time_ago_in_words" data-timestamp="Mon Jul 18 12:21:12 UTC 2011" data-include-seconds="false">18 Jul 14:21</span>
    
Which will be replaced by the client-side javascript with:

    <span class="time_ago_in_words" data-timestamp="Mon Jul 18 12:22:11 UTC 2011" data-include-seconds="false">2 minutes ago</span>

Cacheable Flash Messages
========================

A common issue with caching, is the use of Rails built-in method for displaying short notification-messages to the user: flash[]
If you try rendering flash-messages on a cached page, you will quickly find yourself in a whole lot of trouble.
The message is either not displayed at all, because the request never reaches Rails, or the message is burned into the page, because your cache considers it part of the page and therefore caches it like any other content.

Instead of this approach, cache-wagon will upon every request, store any flash-messages in the users session, ready to be extracted with another request.
cache_wagon provides the read_flash_from_session method which works just like the flash-hash, by returning a hash of all the stored flash-messages, and then clears the messages.
You can then use a second AJAX request to your server, to render these messages out to the client.

To use this feature, all you have to do, is to add the following before_filter to application.rb or any controller that writes flash-messages that needs to be cacheable.

    after_filter :write_flash_to_session

**Here is an example:**

Let's say that we have a HTTP-cached page, that needs to render a dynamic menu that changes depending on the users login-status.
We have already set an AJAX request in place to load this menu from our server on every request.
The server has a js.erb template, where we can simply add the following:

    <%- read_flash_from_session.each do |key, msg| -%>
    display_flash_message("<%= escape_javascript(key.to_s.capitalize) %>", "<%= escape_javascript(msg) %>");
    <%- end -%>

This will call the following piece of javascript, that renders the messages into an empty div on our page:

    function display_flash_message(key, message) {
      $('<p class="notification notification' + key + ' copyPrimary" style="display:none">' + message + '</p>').appendTo('#flash_messages').slideDown("slow");
    }

How you choose to render the flash-messages is completely up to you.

Cacheable Dynamic Block
=======================
Another common issue with caching, is what if your page contains administration links like 'edit' and 'delete' links for each post on a blog and you want to displace these links only to users of relevance (i.e. admins). It is easy to make a check in your code to see if the current user is an administrator and thus allowed to see the links. But what happens when your page is fully cached?

What we do, is to move our role check to a before_filter in the controller instead, to make sure that a user that isn't an administrator can't access the edit/update/destroy actions. And then in the view we display the links with their style attribute set to 'display: none' and show the links using a bit of javascript.

This is what cacheable_dynamic_block can help you with.

**Example:**
Let's use the post-view on a blog as the example

    cacheable_dynamic_block(@post.user.id, :class => 'owner-links') do
      link_to 'edit', edit_admin_post(@post)
    end

Will produce:

    <span class="owner-links" data-owner-id="1" style="display: none;">
      <a href="/posts/2/comments/35/edit">Edit</a>
    </span>
    
Now we need a way to show these links to the right users. In the js.erb template described in the previous step "Cacheable Flash Messages" (that gets loaded from our server on every request), we will call our javascript function that will handle what dynamic blocks to display to the user.

    displayCacheableDynamicBlocks('.owner-links', <%= current_user? ? current_user.id : '' %>, <%= current_user? ? current_user.is_admin? : false %>)