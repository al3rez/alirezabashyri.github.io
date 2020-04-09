---
layout: post
title: Service Objects with ActiveModel and SimpleDelegator
date: 2020-04-09 00:02 +0700
---
![serve](/serve.png)

One of the worst practices in Ruby On Rails is to clutter the Gemfile to do
basics as authentication, authorization, validation, etc. considering the
fact that they are practiced for so long and they can be accomplished using just
the batteries already included in Ruby On Rails; service objects are no
different for which developers often end up using different gems to do such a
simple pattern.

In the favour of simplicity and the fact that I never trust dependencies due to 
to the complexity they bring in I've been using `ActiveModel` and
`SimpleDelegator` to apply service objects through a consistent interface
across the controllers as well as avoiding dependency clutter and here's the results.


# Login case 
Let's consider a user logging in use case;
```
> Users can login with password and email
> Users cannot login with wrong password / email
```

So we have *2 cases* which we have to cover in our implementation to do that I
use `ActiveModel::Validations` to validate the parameters I receive from the
controller and using `save` method to keep interface consistency.


```ruby
class Login
  include ActiveModel::Model

  attr_accessor :email, :password, :user
  validates :email, presence: true
  validates :password, presence: true
  validates :user, presence: {message: "email or password is wrong"}

  def initialize(params)
    super(params.require(:user).permit(:email, :password))
    @user = User.find_by(email: email, status: :active)&.authenticate(password)
  end

  def save
    return if invalid?
    # This will generate and refresh user's token and return it
    Tokenizer.generate_and_refresh!(@user)
  end
end
```
Here if no user found it will set `@user` variable to `nil` and user presence
validation will handle the error which keep us from writing an error handling
part except a message.

And the controller would be like;

```ruby
class LoginController < ApplicationController
  skip_before_action :authenticate
  INCLUDED = %i[positions]

  def create
    login = Login.new(params)

    if (user = login.save)
      render json: user, include: INCLUDED
    else
      render json: login, status: :bad_request
    end
  end

  private

  def serializer
    UserSerializer
  end

  def user_params
    params.require(:user).permit(:email, :password)
  end
end
```

This is already battle-tested in _**production**_ and it works like a charm.

## Approval flow
Let's consider a use case that your flow needs to go through multiple approvals
and each approval stage has its own validations and business logic. There are a
lot of ways to handle this;

### `ActiveRecord` callbacks
It may seem simple solution but you have to write tons of conditions to avoid
conflict between different use cases and hard to debug.

### Contextual validations with `on: :context`
This works just fine but it'll clutter your models with a lot of business logic
which it needs to somewhere else and it's hard to test and modify due to
it's coupled with your model. e.g.

```ruby
validate :order_is_picked_before_packing, on: :packing
```

### Decorators + State machines
State machines needs to be force-feed to developers as they make handling data
logic so convenient but to provide a good interface for your controllers you
cannot treat state machine transitions or guard errors as you best bet you
still need something in-between which in this case is a *decorator* which give
you best of both worlds *validations*, *callbacks*, separation of concern, ease
of testing and etc. In our case we need some conditions are met before going to
another state so the uses cases are;

```
> MVPs cannot be approved before they're submitted
> MVPs cannot be approved more than once by same user
> MVPs cannot be approved unless they have enough approvals from different users
```

Let's take look at the code;
```ruby
class ApprovedMvp < SimpleDelegator
  include ActiveModel::Validations

  def save(user)
    return unless valid?

    if in_state?(:draft)
      errors.add(:minimal_viable_product, "it should be submitted first")
      return
    end

    if approvals.find_by(user: user)
      errors.add(:minimal_viable_product, "already been approved by user")
      return
    else
      approvals.create!(user: user)
    end

    if has_enough_approvals?
      transition_to!(:approved)
    else
      super()
    end
  end
end

```

Here we populate a decorated `MVP` with contextual errors and pass it up to the controller 

```ruby
class UseCases::MinimalViableProductsController < ApplicationController
  INCLUDED = UseCasesController::INCLUDED

  include UseCaseScoped

  def approve
    result = ApprovedMvp.new(minimal_viable_product)

    if result.save(@current_user)
      render json: result.use_case, include: INCLUDED
    else
      render json: result
    end
  end


```

In which will be handled by a `ErrorSerializer` and a concern called
`ActsAsJSONAPI` which I've written to handle different cases of serialization and
error handling without dependencies and a lot of complexity which you can
checkout [here](https://github.com/Netflix/fast_jsonapi/issues/102#issuecomment-559920150).


## Conclusion 
Codes that are written 50 years ago with zero-dependencies are working fine
today and the ones are written today with dependencies may not work tomorrow :)

And don't get me wrong I don't mean to reinvent the wheel but don't be afraid
to build your own tool since each app has different needs and not all
gems/libraries are going to fit yours. Bad cases that you _should_ never
implement by yourself except for practice is encryption or an premature abstraction 
(framework, etc) that's takes your more than 1 day or 2.

I hope you've enjoyed it


<br/>
<hr/>
<br/>
Questions? *Email me at alirezabashiri@fastmail.com*
<br/>
*Sorry I won't add comment section to my blog here.*
