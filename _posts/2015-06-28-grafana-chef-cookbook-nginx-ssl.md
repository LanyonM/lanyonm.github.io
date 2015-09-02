---
layout: post
title: "Grafana Chef Cookbook Wrapper to Override Nginx SSL Template"
description: "How to Use the Chef compile phase to override the Nginx SSL template when using a wrapper cookbook"
category: articles
tags: [chef, data visualization, devops, grafana, monitoring, nginx, operations, ruby, software]
og:
  image: 'grafana-2.png'
  type: 'image/png'
  width: '927'
  height: '362'
comments: true
---

I recently contributed to the overhaul and 2.0 release of the [Grafana Chef cookbook](https://supermarket.chef.io/cookbooks/grafana). It was a nearly complete rewrite of the 1.x version, and many decisions were made along the way about what should (and should not) be included in the effort. The cookbook is designed to be as flexible as possible via attributes and to provide the user with a functional setup using the defaults. The previous version used Nginx as a web server, and it made sense to proxy the new Grafana with Nginx in the default setup.

One of the initially identified tasks was to provide SSL by default within the cookbook, but that proved to be foolish for two reasons: 1) it was well outside the scope of the Grafana cookbook and 2) SSL configs are highly dependent on several diverse factors ranging from web server version to client browser requirements. Creating a default Nginx SSL setup that was flexibly configurable with attributes was duly marked as out of scope, [but not without some discussion](https://github.com/JonathanTron/chef-grafana/issues/47).

For reference, here's the default Nginx template from the Grafana cookbook:

{% highlight ruby %}
template '/etc/nginx/sites-available/grafana' do
  source node['grafana']['nginx']['template']
  cookbook node['grafana']['nginx']['template_cookbook']
  notifies :reload, 'service[nginx]'
  mode '0644'
  owner 'root'
  group 'root'
  variables(
    grafana_port: node['grafana']['ini']['server']['http_port'] || 3000,
    server_name: node['grafana']['webserver_hostname'],
    server_aliases: node['grafana']['webserver_aliases'],
    listen_address: node['grafana']['webserver_listen'],
    listen_port: node['grafana']['webserver_port']
  )
end
{% endhighlight %}


## Overriding The Nginx Conf Template
When defining an Nginx template with SSL enabled, it's helpful to have additional variables passed to the template so info like cert locations can be more flexibly defined. Using `cookbook` and `source` attributes provided by the recipe would allow for the wrapper cookbook to define a new template, but _not_ allow for additional variables to be passed to the template. By taking advantage of Chef's [compile phase](https://docs.chef.io/chef_client.html#the-chef-client-title-run), we can alter the `template['/etc/nginx/sites-available/grafana']` resource to not only use the template of our choosing, but also pass additional attributes to the resource.

The resource definition below is from a Grafana cookbook wrapper recipe:

{% highlight ruby %}
begin
  r = resources(template: '/etc/nginx/sites-available/grafana')
  r.cookbook 'wrapper-cookbook'
  r.source 'default/grafana/nginx.conf.erb'
  r.mode 0644
  r.variables(
    grafana_port: node['grafana']['ini']['server']['http_port'] || 3000,
    server_name: node['grafana']['webserver_hostname'],
    server_aliases: node['grafana']['webserver_aliases'],
    listen_address: node['grafana']['webserver_listen'],
    listen_port: node['grafana']['webserver_port'],
    ssl_cert_dir: node['wrapper-cookbook']['ssl']['cert_dir'],
    ssl_cert_name: node['wrapper-cookbook']['ssl']['cert_name'],
    ssl_key_dir: node['wrapper-cookbook']['ssl']['key_dir'],
    ssl_key_name: node['wrapper-cookbook']['ssl']['key_name'],
    ssl_dhparam_name: node['wrapper-cookbook']['ssl']['dhparam']
  )
  r.notifies :reload, 'service[nginx]', :immediately
  rescue Chef::Exceptions::ResourceNotFound
    Chef::Log.warn 'could not find template to override!'
end
{% endhighlight %}

The recipe is `grafana` and the containing cookbook is `wrapper-cookbook`. When compared to the template within the Grafana cookbook's [`_nginx.rb`](https://github.com/JonathanTron/chef-grafana/blob/v2.0.0/recipes/_nginx.rb#L22-L36), you can see that five additional SSL-related attributes are passed to the template (lines 12-16).

## A Sample Nginx SSL Template
The `nginx.conf.erb` will be dependent on your SSL requirements, but here's an example:

{% highlight erb %}
#
# This file was generated by Chef for <%= node['fqdn'] %>.
# Do not modify this file by hand!
#

upstream grafana {
  server 127.0.0.1:<%= @grafana_port %>;
}

server {
  listen                <%= "#{@listen_address}:" if !@listen_address.nil? && !@listen_address.empty? %><%= @listen_port %>;

  server_name           <%= @server_name %> <%= @server_aliases.join(" ") %>;
  access_log            /var/log/nginx/<%= @server_name %>.access.log;

  ssl on;
  ssl_certificate <%= @ssl_cert_dir %>/<%= @ssl_cert_name %>;
  ssl_certificate_key <%= @ssl_key_dir %>/<%= @ssl_key_name %>;
  ssl_dhparam <%= @ssl_cert_dir %>/<%= @ssl_dhparam_name %>;

  ssl_session_timeout 1d;
  ssl_session_cache shared:SSL:50m;

  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers 'AES256+EECDH:AES256+EDH';
  ssl_prefer_server_ciphers on;

  add_header Strict-Transport-Security max-age=31536000;

  ssl_stapling on;
  ssl_stapling_verify on;

  location / {
    proxy_pass http://grafana;
  }
}
{% endhighlight %}

Your mileage will most certainly vary. If you're looking for suggestions on what SSL configs to use Mozilla has put together a great TLS configuration generator: <https://mozilla.github.io/server-side-tls/ssl-config-generator/>.

## Conclusion
You could argue that this is unnecessary for the Grafana cookbook's `_nginx.rb` given that the recipe is so simple, but it illustrates the power of Chef's compile phase to override the defaults of the cookbooks used.

In addition to a default Nginx setup, the cookbook provides LWRPs for creating datasources, dashboards, organizations, and users. It'll be exciting to see how people use and extend it.

<div class="center">
  <figure>
    <a href="/images/grafana-2.png"><img src="/images/grafana-2.png"></a>
    <figcaption>Sample Dashboard configured with the Grafana cookbook.</figcaption>
  </figure>
</div>