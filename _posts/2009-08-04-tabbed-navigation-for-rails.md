---
layout: post
title: Tabbed Navigation for Rails
---

When I start a new rails app and write a tabbed menu, I always postpone the moment I'll determine how it's going to set the current tab. And when it comes to it, I usually do quick and dirty stuff with the controller name, the @category.name or whatever. But last time, facing a step by step form you could navigate through with tabs, I thought "enough of it I need a solution once and for all".

Sooo, looking at plugins first <a href="http://wiki.github.com/paolodona/rails-widgets/tabnav">tabnav</a> seemed to be the reference. But I wanted something i could easily tweek and understand, and this was just overkill for me needs.

Then I looked at the <a href="http://www.therailsway.com/2007/6/4/free-for-all-tab-helper">free for all tab helper</a> which has some very interesting solutions, the most elegant being the one relying only on css and the body tag (no logic involved). But that wasn't flexible enough, they were all making the assumption the tabs would switch controllers.

So having to get my hands dirty I first refreshed my memory with my <a href="http://railscasts.com/episodes/101-refactoring-out-helper-object">favourite source of hints</a> and came up with what I believe is a very elegant and flexible solution.

``` ruby
#contenth
  %ol#steps_menu
    - tabs_for :step => active_step do |t|
      = t.link_to "1. start", :step=> "1start"
      = t.link_to "2. price", :step=> "2price"
      = t.link_to "3. bill", :step=> "3bill"
      = t.link_to "4. payment", :step=> "4payment"
      = t.link_to "5. end", :step=> "5end"

  =render :partial => "/admin/orders/steps/step_#{active_step}"
  ```

And the Helper :

``` ruby
module TabsHelper

  # takes the block
  def tabs_for(current_tab, &block)
    yield Tab.new(current_tab, self)
  end

  class Tab
    def initialize(current_tab, template)
      @current_tab = current_tab
      @template = template
    end

    def link_to(*args)
      "<li #{active_class(*args)}>#{@template.link_to(*args)}</li>"
    end

  private
    def active_class(*args)
      if @current_tab.all? { |k, v| args[1][k] == v  }
        "class = 'active'"
      end
    end
  end #class Tab

end
```

This is minimalistic but you can see that it is very easily extendable. The main idea is that you are sendind to the tab class a matcher that is going to set the current tab.

My matcher was a bit complex as you can see below(it also sets the partial to render), but keep in mind that you could do exactly the same like this :

tab_for :controller => controller_name do |t| <br />...

and it works with no further logic.

``` ruby
module OrdersHelper

  # helper to determine the active order step given :
  # 1. a params[:step]
  # 2. if none, according to the order's state
  def active_step
    @active_step ||= if params[:step]
        params[:step]
      elsif %(opened).include? @order.state
        "1start"
      elsif %(submitted).include? @order.state
        "2price"
      elsif %(evaluated).include? @order.state
        "3bill"
      elsif @order.state=="billed"
        "4payment"
      else
        "5end"
      end
  end
end
```

What I appreciate most is that the logic for matching the tab is kept seperate ( it only relies on the params inside the links ) and the "tab_for do" syntax is very railsish & familiar.
