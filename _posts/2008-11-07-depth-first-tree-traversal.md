---
layout: post
title: depth first tree traversal
---

I started a little library called <a href="http://github.com/charly/raaws/tree/master">RAAWS for Ruby Associate Amazon Web Services</a>. It's in a very early stage of development, so don't use it yet.

Anyway fiddling around with amazon api responses, I often felt like having a clear look at the xml directly in my a rails view, but found nowhere a good depth-first tree traversal method in any of the libraries (hpricot, libxml, nokogiri, rexm...ha no) to help me spit it out in a readable html list.

So after a little struggle :

``` ruby
require "rubygems"
require "xml/libxml"
#require "active_support"



def traverse(node)
  children = node.children.select {|a| a.element?}
  txt = "<h1>HTML output</h1><ul>"
  last_depth, depth = 1, 1

  while child = children.shift
    depth = child.path.split("/").size.to_i - 2

    if depth > last_depth
      txt << "\n<ul>"
    elsif depth < last_depth
      txt << "</ul>\n" * (last_depth - depth)
    end

    txt << "<li>#{child.name} : <i>#{child.content if child.children.all? {|a| a.text?}}</i></li>\n"

    a = child.children.select {|c| c.element?}
    unless a.empty?
      children.unshift(a).flatten!
    end

    last_depth = depth
  end

  open("output.html", "w") do |f|
    f.write txt << "</ul>"
  end
end


xml = XML::Document.file("chaplin.xml")
# p xml.root.namespaces
traverse(xml.root)
```


If anyone has a better way, please let me know, specially with node.path.split('/') to get the nodes depth, it' really a dirty hack.
