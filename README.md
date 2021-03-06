# Adyengo

This is an Adyen app for Django. It provides an easy way to integrate the Adyen
payment system into a Django project. Regular, oneclick and recurring payments
are supported. All communication to and from Adyen is logged.

The project is however far from perfect and it hasn't been used in production
yet. There might be bugs, so be aware if you want to use this. Also, if you
use this app, you should still know how Adyen works, the app just simplifies
the process. However, occasionally the implementation documentation will
explain what happens under the hood. Sometimes it will refer to the Adyen
documentation for futher reading.


## Development

As said before, the project is far from perfect. Not all Adyen features are
implemented and the system hasn't even been tested thoroughly in production.
Since Adyen implementation is not straightforward and Adyen is a quite popular
payment service provider and Django is a popular framework, I hope people will
find use for it.

If you want to contribute, you're very welcome. Contact me for any questions.


## Installation

I haven't made a pip install for it yet, so you'll have to manually add it to
your project. Don't forget to install the requirements in `requirements.txt`.


## Implementation

### Settings

To start out you should add some settings to your main `settings.py`.

#### Required settings

* `ADYENGO_MERCHANT_ACCOUNT` - Your Adyen merchant account.
* `ADYENGO_SHARED_SECRET` - The shared secret used to validate the communication with
    Adyen.
* `ADYENGO_MODE` - Use `test` to use the Adyen test environment or `live` to use the
    live environment.

#### Required for Recurring Payments

* `WEB_SERVICES_USERNAME` - The username for the Adyen Web Services SOAP API.
* `WEB_SERVICES_PASSWORD` - The password for the Adyen Web Services SOAP API.
* `WEB_SERVICES_PUBLIC_KEY` - The public key for the Adyen Web Services SOAP API.

#### Optional settings

* `DEFAULT_SKIN_CODE` - The default skin code. Can always be overwritten.
* `DEFAULT_CURRENCY_CODE` - The default currency code. Can always be
    overwritten.

### Usage

There are two ways to do a payment through Adyen. One is to forward the
customer to the **Adyen Hosted Payment Pages (HPP)** and the other is to do a
**Recurring Payment through Adyen's API**.

Adyengo reflects Adyen's interface in regular Django models and managers.
The main model to setup a payment is the `Session` model. Depending on what
kind of payment, you fill the fields of the `Session` model.

#### The `Session` model

Most of the fields in the `Session` model reflect a field in the Adyen
interface. The only difference is that Adyen uses `camelCase` and Adyengo uses
`under_scores`. So a field in the Adyen interface like `merchantReference` will
become `merchant_reference` in Adyengo. We will refer to these fields as "Adyen
specific fields", other fields we'll call "Adyengo specific fields". When a
field in the `Session` model is an Adyen specific field it will be pointed out.
In this case you can get more information about the field in the Adyen
Integration Manual.

Besides the Adyen specific fields, the `Session` model has some Adyengo
specific fields for internal logic. The most important one is the
`session_type` field. This field decides what kind of payment the session
represents.

The choices for this field are defined in the `constants.py` which you can
import using `from adyengo import constants`. Then you can choose one of the
following:

* `constants.SESSION_TYPE_HPP_REGULAR` - For a regular payment.
* `constants.SESSION_TYPE_HPP_RECURRING` - To setup a recurring contract,
    or to do a "oneclick" recurring payment.
* `constants.SESSION_TYPE_API_RECURRING` - For a recurring payment.

Now let's set up a Regular payment so you get a feeling of how it works.

#### Regular Payment

##### Fields

To set up a regular payment, you should set the `session_type` field to
`constants.SESSION_TYPE_HPP_REGULAR` and provide the following Adyen specific
fields:

* `merchant_reference` - Unique id for the session.
* `payment_amount` - Amount for the payment in cents.
* `currency_code` - The currency code for the currency the amount is in.
    Choices are listed in the constants starting with `CURRENCY_CODE_`, for
    example `CURRENCY_CODE_EUR` for Euro.
* `ship_before_date` - The date the product should be shipped, provide as a
    Python date object.
* `skin_code` - The skin code. If you set a default skin code using the
    `DEFAULT_SKIN_CODE` settings, you can exclude this field. If you provide it
    anyway, it will overwrite the default skin code.
* `session_validity` - The date the session expires, provide as a Python date
    object.

Optional Adyen specific fields:

* `shopper_reference` - Unique id for the customer.
* `shopper_email` - The customer's email address.

There's one Adyengo specific field you should provide, `page_type`. This field
decides to what kind of page the user gets forwarded. The choices are defined
in the constants:

* `constants.PAGE_TYPE_MULTIPLE` - For a multiple pages flow.
* `constants.PAGE_TYPE_SINGLE` - For a more modern single page app.
* `constants.PAGE_TYPE_SKIP` - To skip the page flow. For this choice,
    precicely one allowed payment method should be provided. We'll describe
    later how to do that.

##### Logging

When you've set up the `Session` model, you can `save()` it. This will
automatically turn into a log for you which you can consult in the Django
Admin.

##### Params

To forward the user to adyen, you should have the session set up and put the
parameters in hidden form fields on the page you want to forward the user. You 
can then either have the user forward himself by clicking a submit button, or
automatically forward the user by submitting the form with javascript.

To get the parameters for the form, use the `params()` method on the `Session`
model. To get the url the form should post to, use the `url()` method.

##### Example

**view.py**

    from django.shortcuts import render
    from adyengo.models import Session
    from adyengo import constants
    from django.utils import timezone

    def forward_to_adyen(request):

        tomorrow = (timezone.now() + timezone.timedelta(days=1))

        s = Session(
            session_type=constants.SESSION_TYPE_HPP_REGULAR,
            merchant_reference=51391,
            payment_amount=1000,
            currency_code=constants.CURRENCY_CODE_EUR,
            ship_before_date=tomorrow,
            skin_code='Nl0r8s5C',
            session_validity=tomorrow,
            shopper_reference=13154,
            shopper_email='shopper@example.com',
            page_type=constants.PAGE_TYPE_MULTIPLE
        )
        s.save()

        url = s.url()
        params = s.params()

        return render(request, 'forward.html', {
            'url': url,
            'params': params
        })

**forward.html**

    <form method="POST" action="{{ url }}">
        {% for var, value in params.items %}
            <input type="hidden" name="{{ var }}" value="{{ value }}" />
        {% endfor %}
    </form>

    <script>
        document.forms[0].submit() // automatically submit the form
    </script>


#### Oneclick Recurring Payment

This readme isn't finished yet. The code however is usable, so don't hesitate
to try it out.

### Adyengo Test screens

Adyengo has some test screens, which makes it easy to test different
kind of requests to Adyen. You can enable them by adding them to your main
`urls.py`:

    import adyengo

    urlpatterns = patterns('',
        # Here your regular urlpatterns
    )

    if settings.DEBUG:
        urlpatterns += patterns('',
            url(r'^adyengo-test/', include(adyengo.site.urls))
        )
