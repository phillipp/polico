# polico

PORO authorization in Rails. Super-object-oriented. Less magic, more objects.

## How to use

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

Create a new class (yes, for every class you want to authorize, you add a new class).

```
class Polico::ContractPolicy < Polico::Policy
  def can_update?
    user.project_manager? || contract.unsigned?
  end
end
```

3. That's it.

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

## Stuff the policy can do for you

Ah, you noticed it already: what if the user in the policy is not existing? If the user is not signed in? How do you catch that?

```
class Polico::ContractPolicy < Polico::Policy
  # Use allow_guests to create contracts for not-signed-in users
  allow_guests to: [ :create ]

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

  # If only signed in users should create
  def can_create?
    user.present?
  end

  # or, shorter:
  allow_everybody_to :create # Guests can create
  allow :create # Only users can create
end
```
