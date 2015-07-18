# polico

Policies, including authorization, in Rails. Less magic, more objects. Oriented to policies, reusable.

polico is more of a design pattern than a framework, because it is no magic and just PORO.

Two reasons for this:

* CanCan etc. ability-files get huge fast
* Authorization get very complex. Add "very"s to the sentence as you see fit.
* Access control is duplicated a lot of times and should be more reusable

And I just love small objects.

## How to use

A frequent use case would be authorization in a controller.

1. Add a check to your Rails before_filter to check who can do what:

```
class ContractController < ApplicationController
  before_filter :authorize, only: [ :edit, :update ]
  
  def edit
    @contact = Contract.find(params[:id])
  end

  def update
    @contact = Contract.find(params[:id])

    if @contact.update_attributes(params[:contract])
      redirect_to contact_path(@contract), notice: 'Yay, updated the contract!'
    else
      render 'edit'
    end
  end

  private

  # The can! will raise a Polico::Unauthorized exception if the user is not
  # authorized to update the contract. Use can? if you want to ask a question.
  def authorize
    can!(:update, @contract)
  end
end
```

2. Add your policy

Create a new class (yes, for every class you want to authorize, you add a new class) in `app/lib/polico/contract_policy.rb`

```
class Polico::ContractPolicy < Polico::Policy
  def can_update?
    user.project_manager? || contract.unsigned?
  end
end
```

3. That's it. Now create more policies

# How to test your Policies

```
require 'spec_helper'

describe 'Polico::Contract' do
  subject(:policy) { Polico::ContractPolicy.new(contract) }
  let(:contract)   { FactoryGirl.build(:contract) }
  let(:user)       { FactoryGirl.build(:project_manager) }

  it 'allows the project manager to update the contract' do
    expect(policy).to permit_to(:update, contract).as(user)
  end
end
```

## Stuff you can do with policies

Let's look at another example. This would be useful if you wanted to check if it's okay to charge a customers credit card. And you can delegate checking a credit card to the policy in a validation!

```
# Checks if the user can create a charge for on credit card
class Polico::CreditCardPolicy < Polico::Policy
  def can_charge?
    !credit_card.fraudulent? && no_outstanding_invoices? 
  end

  private

  def no_outstanding_invoices?
    user.invoices.where(paid: false).none?
  end
end
```

And this is totally reusable, for example in your billing run:

```
  def bill_customer(customer)
    if Polico::CheckPolicy.new(customer.default_credit_card, :charge, customer).allowed?
      create_charge(customer)
    else
      Notification.billing_failed(customer).deliver
    end
  end
```

Or you could have a policy if a user can order a specific item and display an error message in the cart. Or check if a visitor (via a request object) is allowed to sign up by the IP address. And much more!

## Stuff you can do with Polico::Policy

Ah, you noticed it already: what if the user in the policy is not existing? If the user is not signed in? Well, **by default guests can't do a thing**. The contract will always be populated with the `user` object (that is the `current_user`) from the controller and the subject of the policy. The subject is accessible by `subject` or the class name of the object (without the namespace), like `contract` for the `Contract` class or `record` for `Nameserver::Record`.

```
class Polico::ContractPolicy < Polico::Policy
  # Use allow_anyone_to to create contracts, even not-signed-in users
  allow_anyone_to to: [ :create ]

  def can_update?
    user.project_manager? || contract.unsigned?
  end
end
```

**By default, nothing is allowed.** This makes sure that you don't grant "access by typo". So you have to always specify what is allowed:

```
class Polico::ContractPolicy < Polico::Policy
  # Guests can create, too:
  def can_create?
    true
  end

  # Only signed in users should update
  def can_update?
    user.present?
  end

  # or, shorter:
  allow_anyone_to :create # Guests can create
  allow_users_to :update # Only users can create
end
```

## Roles?

There are no roles included. And that's on purpose. I agree that it would certainly be better for some use cases if the role would specify what the user can do. But in my use case, the object and it's properties have more influence on what the user can and can't do that the role of the user, because all users are the same (like, in a control panel).

But you can add roles yourself quite easily:

```
class User < ActiveRecord::Base
  def roles
    read_attribute(:roles).split(',')
  end

  def has_role?(role_name)
    roles.include?(role_name.to_s)
  end
end
```

You can now use the `.has_role?(:project_manager)` in your policy:

```
class Polico::ContractPolicy < Polico::Policy
  allow_anyone_to :create

  def can_update?
    user.has_role?(:project_manager) || user.has_role?(:legal_counsel)
  end
end
```
