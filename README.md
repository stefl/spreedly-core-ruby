SpreedlyCore
======

spreedly-core-ruby is a Ruby library for accessing the [Spreedly Core API](https://spreedlycore.com/). Spreedly Core is a Software-as-a-Service billing solution that serves two major functions for companies and developers. 

* First, it removes your [PCI Compliance](https://www.pcisecuritystandards.org/) requirement by pushing the card data handling and storage outside of your application. This is possible by having your customers POST their credit card info to the Spreedly Core service while embedding a transparent redirect URL back to your application (see "Submit payment form" on [the quick start guide](https://spreedlycore.com/manual/quickstart)). 
* Second, it removes any possibility of your gateway locking you in by owning your customer billing data (yes, this happens). By allowing you to charge any card against whatever gateways you as a company have signed up for, you retain all of your customer data and can switch between gateways as you please. Also, expanding internationally won't require an additional technical integration with yet another gateway.

Quickstart
----------
Head over to the [Spreedly Core Website](https://www.spreedlycore.com) to sign up for an account. It's free to get started and play with test gateways/transactions using our specified test card data.

RubyGems:

    export SPREEDLYCORE_API_LOGIN=your_login_here
    export SPREEDLYCORE_API_SECRET=your_secret_here
    gem install spreedly-core-ruby
    irb
    require 'rubygems'
    require 'spreedly-core-ruby'
    SpreedlyCore.configure

The first thing we'll need to do is set up a test gateway that we can run transactions against. Then, we'll tell the gem to use the newly created gateway for all future calls.

    tg = SpreedlyCore::TestGateway.get_or_create
    tg.use!

Now that you have a test gateway set up, we'll need to set up your payment form to post the credit card data directly to Spreedly Core. Spreedly Core will receive your customer's credit card data, and immediately transfer them back to the location you define inside the web payments form. The user won't know that they're being taken off site to record to the card data, and you as the developer will be left with a token identifier. The token identifier is used to make your charges against, and to access the customer's non-sensitive billing information.

    <form action="https://spreedlycore.com/v1/payment_methods" method="POST">
	    <fieldset>
	        <input name="redirect_url" type="hidden" value="http://example.com/transparent_redirect_complete" />
	        <input name="api_login" type="hidden" value="Ll6fAtoVSTyVMlJEmtpoJV8Shw5" />
	        <label for="credit_card_first_name">First name</label>
	        <input id="credit_card_first_name" name="credit_card[first_name]" type="text" />
	
	        <label for="credit_card_last_name">Last name</label>
	        <input id="credit_card_last_name" name="credit_card[last_name]" type="text" />
	
	        <label for="credit_card_number">Card Number</label>
	        <input id="credit_card_number" name="credit_card[number]" type="text" />
	
	        <label for="credit_card_verification_value">Security Code</label>
	        <input id="credit_card_verification_value" name="credit_card[verification_value]" type="text" />
	
	        <label for="credit_card_month">Expires on</label>
	        <input id="credit_card_month" name="credit_card[month]" type="text" />
	        <input id="credit_card_year" name="credit_card[year]" type="text" />
	
	        <button type='submit'>Submit Payment</button>
	    </fieldset>
	</form>

Take special note of the **api_login** and **redirect_url** params hidden in the form, as Spreedly Core will use both of these fields to authenticate the developer's account and to send the customer back to the right location in your app.

A note about test card data
----------------
If you've just signed up and have not entered your billing information (or selected a Heroku paid plan), you will only be permitted to deal with test credit card data. Furthermore, we will outright reject any card data that doesn't correspond to the 8 numbers listed below.

DO NOT USE REAL CREDIT CARD DATA UNTIL YOUR APPLICATION IS LIVE.

* **Visa**
	* Good Card - 4111111111111111
	* Failed Card - 4012888888881881
* **MasterCard**
	* Good Card - 5555555555554444
	* Failed Card - 5105105105105100
* **American Express**
	* Good Card - 378282246310005
	* Failed Card - 371449635398431
* **Discover**
	* Failed Card - 6011111111111117
	* Failed Card - 6011000990139424

Once you've created your web form and submitted one of the test cards above, you should be returned to your app with a token identifier by which to identify your newly created payment method. Let's go ahead and look up that payment method by the token returned to your app, and we'll charge $5.50 to it.
 
    payment_token = 'abc123' # extracted from the URL params
    payment_method = SpreedlyCore::PaymentMethod.find(payment_token)
    if payment_method.valid?
      purchase_transaction = payment_method.purchase(550)
      purchase_transaction.succeeded? # true
    else
      flash[:notice] = "Woops!\n" + payment_method.errors.join("\n")
    end
    
One final point to take note of is that Spreedly Core does no validation of the information passed in by the customer. We simply return them back to your application, and it's up to you to check for any errors in the payment method before charging against it.

Saving Payment Methods
----------
Spreedly Core allows you to retain payment methods provided by your customer for future use. In general, removing the friction from your checkout process is one of the best things you can do for your application, and using Spreedly Core will allow you to avoid making your customer input their payment details for every purchase.

    payment_token = 'abc123' # extracted from the URL params
    payment_method = SpreedlyCore::PaymentMethod.find(payment_token)
    if payment_method.valid?
      puts "Retaining payment token #{payment_token}"
      retain_transaction = payment_method.retain
      retain_transaction.succeeded? # true
    end
    
Payment methods that you no longer want to retain can be redacted from Spreedly Core. Spreedly Core only charges for retained payment methods.

    payment_token = 'abc123' # extracted from the URL params
    payment_method = SpreedlyCore::PaymentMethod.find(payment_token)
    redact_transaction = payment_method.redact
    redact_transaction.succeeded? # true
    
Usage Overview
----------
Make a purchase against a payment method

    purchase_transaction = payment_method.purchase(1245)
    
Retain a payment method for future use

    redact_transaction = payment_method.retain
    
Redact a previously retained payment method:

    redact_transaction = payment_method.redact

Make an authorize request against a payment method, then capture the payment

    authorize = payment_method.authorize(100)
    authorize.succeeded? # true
    capture = authorize.capture(50) # Capture only half of the authorized amount
    capture.succeeded? # true

    authorize = payment_method.authorize(100)
    authorize.succeeded? # true
    authorized.capture # Capture the full amount
    capture.succeeded? # true
    
Void a previous purchase:

    purchase_transaction.void # void the purchase

Credit (refund) a previous purchase:

    purchase_transaction = payment_method.purchase(100) # make a purchase
    purchase_transaction.credit
    purchase_transaction.succeeded? # true 

Credit part of a previous purchase:

    purchase_transaction = payment_method.purchase(100) # make a purchase
    purchase_transaction.credit(50) # provide a partial credit
    purchase_transaction.succeeded? # true 


Handling Exceptions
--------
There are 3 types of exceptions which can be raised by the library:

1. SpreedlyCore::TimeOutError is raised if communication with Spreedly Core takes longer than 10 seconds     
2. SpreedlyCore::InvalidResponse is raised when the response code is unexpected (I.E. we expect a HTTP response code of 200 bunt instead got a 500) or if the response does not contain an expected attribute. For example, the response from retaining a payment method should contain an XML attribute of "transaction". If this is not found (for example a HTTP response 404 or 500 is returned), then an InvalidResponse is raised.
3. SpreedlyCore::UnprocessableRequest is raised when the response code is 422. This denotes a validation error where one or more of the data fields submitted were not valid, or the whole record was unable to be saved/updated. Inspection of the exception message will give an explanation of the issue.

      
Each of TimeOutError, InvalidResponse, and UnprocessableRequest subclass SpreedlyCore::Error.

For example, let's look up a payment method that does not exist:

    begin
      payment_method = SpreedlyCore::PaymentMethod.find("NOT-FOUND")
    rescue SpreedlyCore::InvalidResponse => e
      puts "Record does not exist"
      puts e.inspect
    end  

   
Additional Field Validation
----------
The Spreely Core API provides validation of the credit card number, security code, and
first and last name. In most cases this is enough; however, sometimes you may want to
enforce the full billing information as well. This can be accomplished via the following:

    SpreedlyCore.configure
    SpreedlyCore::PaymentMethod.additional_required_cc_fields :address1, :city, :state, :zip
    master_card_data = SpreedlyCore::TestHelper.cc_data(:master)
    token = SpreedlyCore::PaymentMethod.create_test_token(master_card_data)
    payment_method = SpreedlyCore::PaymentMethod.find(token)
    payment_method.valid? # false
    payment_method.errors # ["Address1 can't be blank", "City can't be blank", "State can't be blank", "Zip can't be blank"]
    master_card_data = SpreedlyCore::TestHelper.cc_data(:master, :credit_card => {:address1 => "742 Evergreen Terrace", :city => "Springfield", :state => "IL", 62701})
    payment_method = SpreedlyCore::PaymentMethod.find(token)
    payment_method.valid? # returns true
    payment_method.errors # []

   
Configuring SpreedlyCore for use in production (Rails example)
----------
When you're ready for primetime, you'll need to complete a couple more steps to start processing real transactions.

1. First, you'll need to get your business (or personal) payment details on file with Spreedly Core so that we can collect transaction and card retention fees. For those of you using Heroku, simply change your Spreedly Core addon to the paid tier.
2. Second, you'll need to acquire a gateway that you can plug into the back of Spreedly Core. Any of the major players will work, and you're not at risk of lock-in because Spreedly Core happily plays middle man. Please consult our [list of supported gateways](https://www.spreedlycore.com/manual/gateways) to see exactly what information you'll need to pass to Spreedly Core when creating your gateway profile.

For this example, I will be using an Authorize.net account that only has a login and password credential.

    authorize_credentials = {:login => 'my_authorize_login', :password => 'my_authorize_password'}
    SpreedlyCore.configure
    gateway = SpreedlyCore::Gateway.create(authorize_credentials)
    puts "Authorize.net gateway token is #{gateway.token}"
    gateway.use!
    
For most users, you will start off using only 1 gateway token, and as such can configure an additional environment variable to hold your gateway token. In addition to the previous environment variables, the SpreedlyCore.configure method will also look for a SPREEDLYCORE_GATEWAY_TOKEN environment value.

	# create an initializer at config/initializers/spreedly_core.rb
    # values already set for ENV['SPREEDLYCORE_API_LOGIN'], ENV['SPREEDLYCORE_API_SECRET'], and ENV['SPREEDLYCORE_GATEWAY_TOKEN']
    SpreedlyCore.configure
    
For those using multiple gateway tokens, there is a class variable that holds the active gateway token. Before running any sort of transaction against a payment method, you'll need to set the gateway token that you wish to charge against.

    SpreedlyCore.configure
    
    SpreedlyCore.gateway_token(paypal_gateway_token)
    SpreedlyCore::PaymentMethod.find(pm_token).purchase(550)
    
    SpreedlyCore.gateway_token(authorize_gateway_token)
    SpreedlyCore::PaymentMethod.find(pm_token).purchase(2885)
    
If you wish to require additional credit card fields, the initializer is the best place to set this up.

    SpreedlyCore.configure
    SpreedlyCore::PaymentMethod.additional_required_cc_fields :address1, :city, :state, :zip  

Contributing
------------

Once you've made your commits:

1. [Fork](http://help.github.com/forking/) SpreedlyCore
2. Create a topic branch - `git checkout -b my_branch`
3. Push to your branch - `git push origin my_branch`
4. Create a [Pull Request](http://help.github.com/pull-requests/) from your branch
5. Profit! 

