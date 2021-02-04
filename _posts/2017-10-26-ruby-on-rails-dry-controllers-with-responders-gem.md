---
layout: post
title: "Ruby on Rails: DRY controllers with responders gem"
categories: ruby
date: 2017-10-26 12:00:00 -0800
---
Note (Feb 4, 2021): Over time I've become increasingly convinced that responders, while decreasing the amount of boilerplate in your controller code, hurt more than they help. Focusing on decreasing the amount of controller code you write in general is a better idea. Rails' magic is excellent most of the time, but there's a reason that Rails removed this concept and extracted it into its own gem. Now that I've given you my own opinion, feel free to act on the contents of this post as you see fit!

Rails' ability to generate scaffolding is nice, but the way the resulting controllers are structured seems to abandon some part of the 'thin controller, fat model' ideology (that said, fat models are still not great). Generating a scaffold for a `User` model gives us a controller which handles action responses in this way:

~~~~ruby
# app/controllers/users_controller.rb

def create
  @user = User.new(user_params)

  respond_to do |format|
    if @user.save
      format.html { redirect_to @user, notice: 'User was successfully created.' }
      format.json { render :show, status: :created, location: @user }
    else
      format.html { render :new }
      format.json { render json: @user.errors, status: :unprocessable_entity }
    end
  end
end
~~~~

This action alone occupies 10 lines of the entire file's 74. While it might be easy to read and understand what is going on, it seems like there's a huge opportunity to DRY things up. Not only does this look a unnecessarily large immediately after generating the scaffold, it only gets worse as code complexity increases. Handling responses can quickly become more complicated than "if the record is saved respond like this, otherwise respond like this". While this could be a sign that your controller (and by extension, everything else) could be in need of refactoring, it could only benefit us to have an easier way to dictate how we respond.

This is where the [responders gem](https://github.com/plataformatec/responders) comes in handy. The author has bundled a controller generator with the gem, which can be activated by adding the following line to `config/application.rb`:

~~~~ruby
# config/application.rb

config.app_generators.scaffold_controller :responders_controller
~~~~

Let's take a look at the same controller action, but using our new scaffold generator:

~~~~ruby
# app/controllers/users_controller.rb

respond_to :html

def create
  @user = User.new(user_params)
  @user.save
  respond_with(@user)
end
~~~~

This result is much clearer. At the top of the controller, we define a standard response format, in this case just `:html`, but just as easily a list like `:html, :json, :js`. The `responders` generator leaves off the `json` reponse by default, but you only need to add `:json` to the `respond_to` call. The action itself is now 3 lines long - set up the `User`, save it, and respond with the `User` we just saved. `respond_with` responds based on the state of `@user`. This way, we avoid the biggest problem with the default layout - the if/else causing the code to repeat itself.

Initially you might notice we've lost a bit of functionality, but nothing we can't add back if we do need it. It will respond with whatever formats we've defined using `respond_to`, exactly as it would have previously. If we'd like to include notice messages like before, we can do that - let's add the `:json` response back at the same time:

~~~~ruby
# app/controllers/users_controller.rb

respond_to :html, :json
responders :flash
~~~~

~~~~yml
# config/locales/en.yml

en:
  flash:
    actions:
      create:
        notice: "%{resource_name} was successfully created."
        alert: "%{resource_name} could not be created."
      update:
        notice: "%{resource_name} was successfully updated."
        alert: "%{resource_name} could not be updated."
      destroy:
        notice: "%{resource_name} was successfully destroyed."
        alert: "%{resource_name} could not be destroyed."
~~~~

At first glance, this might seem like a lot of work - but this only needs to be done once. This is more convenient though - we now have one central place to change all of these messages, which apply to all models. If we want to override this for a particular model, we can do that too:

~~~~yml
# config/locales/en.yml

flash:
  users:
    create:
      notice: "The account was succcessfully created."
      alert: "The account could not be created."
~~~~

With these changes we've made, our controllers are much smaller - `users_controller.rb` was 74 lines, now 48 - with no loss in functionality.

If you'd like to provide a custom redirect path, that is as simple as providing a `location`:

~~~~ruby
# app/controllers/users_controller.rb

respond_with(@user, location: users_path)
~~~~

It's as simple as that. May your bugs be squashed and your controllers, dry. Happy hacking!
