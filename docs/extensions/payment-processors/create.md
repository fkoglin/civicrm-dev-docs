# Create a Payment Processor Extension

This page provides a step-by-step guide to creating a payment processor extension.

It uses an example of one created at the University of California, Merced. It is a basic payment processor which works like this:

1.  Civicrm collects basic information (name, email address,
    amount, etc.) and passes this onto the payment processor and
    redirects to the payment processors website.
2.  The payment processor collects the credit card and any additional
    information and processes the payment.
3.  The payment processor redirects the user back to civicrm at a
    specific url.
4.  The extension then processes the return information and completes
    the payment.

## Research

You need to check your processor's documentation and understand the flow
of the processor's process and compare it to existing processors 

Factors you should take note of are:

-   Does your processor require the user to interact directly with
    forms/pages on their servers (like PayPal Express), or are are data
    fields collected by CiviContrib form and submitted "behind the
    scenes"?
-   What fields are required?
-   What authentication method(s) are used between Civi and the payment
    processor servers?
-   In what format is data submitted (soap/xml, arrays...)?
-   What are the steps in completing a transaction (e.g. simple POST and
    response, vs multi-step sequence...). Are transaction results
    returned real-time (PayPal Pro/Express and Moneris) - or
    posted back in a separate process (PayPal Standard w/ IPN)?

Note that none of the plugins distributed with CiviCRM use a model where
the donor's credit card info is stored in the CiviCRM site's database.
For PayPal Pro, Authorize.net and PayJunction - Credit card numbers, exp
date and security codes are entered on the CiviCRM contribution page and
immediately passed to the processor / not saved. For PayPal Std, Google
Checkout - the info is entered at the processors' site.


## Setup

1. Use civix to [generate an skeleton extension](/extensions/civix.md#generate-module)

1. Identify the processor type {:#type}
   
   Read about [processor types](/extensions/payment-processors/types.md) to find out which type you have.

1. Add the processor to the database

    To insert a new payment processor plugin, a new entry must be added to
    the `civicrm_payment_processor_type` table.

1. Store any function files from your payment processor

    Create an appropriately named folder in the 'packages' directory for any
    files provided by your payment processor which have functions you need

1. Edit the `info.xml` file.

    Edit your [info.xml file](/extensions/info-xml.md) and add the [typeInfo section](/extensions/info-xml.md#typeInfo) with all relevant child elements.



### Create the Default Payment Processor File

Depending on your billing mode there are different considerations. The file
will live in `CRM/Core/Payment` and have the same name as entered into
your `processor_type` table.


### Create Initial Processing File

In our example our file name is UCMPaymentCollection so the name of the
file we are going to create is UCMPaymentCollection.php

It should have this basic template.

**UCMPaymentCollection.php**

```php
<?php

class edu_ucmerced_payment_ucmpaymentcollection extends CRM_Core_Payment {

  /**
   * mode of operation: live or test
   *
   * @var object
   * @static
   */
  static protected $_mode = null;

  /**
   * Constructor
   *
   * @param string $mode the mode of operation: live or test
   *
   * @return void
   */
  function __construct( $mode, &$paymentProcessor ) {
    $this->_mode             = $mode;
    $this->_paymentProcessor = $paymentProcessor;
    $this->_processorName    = ts('UC Merced Payment Collection');
  }

  /**
   * This function checks to see if we have the right config values
   *
   * @return string the error message if any
   * @public
   */
  function checkConfig( ) {
    $config = CRM_Core_Config::singleton();

    $error = array();

    if (empty($this->_paymentProcessor['user_name'])) {
      $error[] = ts('The "Bill To ID" is not set in the Administer CiviCRM Payment Processor.');
    }

    if (!empty($error)) {
      return implode('<p>', $error);
    }
    else {
      return NULL;
    }
  }

  /**
   * Sets appropriate parameters for checking out to UCM Payment Collection
   *
   * @param array $params  name value pair of contribution datat
   *
   * @return void
   * @access public
   *
   */
  function doPayment(&$params, $component) {

  }
}
```


There are 4 areas that need changing for your specific processor.

The class name needs to match to your chosen class name (directory of
the extension)

```php
class edu_ucmerced_payment_ucmpaymentcollection extends CRM_Core_Payment {
```

The processor name needs to match the name of your processor.

```php
function __construct( $mode, &$paymentProcessor ) {
  $this->_mode             = $mode;
  $this->_paymentProcessor = $paymentProcessor;
  $this->_processorName    = ts('UC Merced Payment Collection');
}
```


You need to process the data given to you in the doTransferCheckout()
method. Here is an example of how it's done in this processor


**UCMPaymentCollection.php-doTransferCheckout**

```php
function doPayment( &$params, $component ) {
    // Start building our paramaters.
    // We get this from the user_name field even though in our info.xml file we specified it was called "Purchase Item ID"
    $UCMPaymentCollectionParams['purchaseItemId'] = $this->_paymentProcessor['user_name'];
    $billID = array(
      $params['invoiceID'],
      $params['qfKey'],
      $params['contactID'],
      $params['contributionID'],
      $params['contributionTypeID'],
      $params['eventID'],
      $params['participantID'],
      $params['membershipID'],
    );
    $UCMPaymentCollectionParams['billid'] =  implode('-', $billID);
    $UCMPaymentCollectionParams['amount'] = $params['amount'];
    if (isset($params['first_name']) || isset($params['last_name'])) {
      $UCMPaymentCollectionParams['bill_name'] = $params['first_name'] . ' ' . $params['last_name'];
    }

    if (isset($params['street_address-1'])) {
      $UCMPaymentCollectionParams['bill_addr1'] = $params['street_address-1'];
    }

    if (isset($params['city-1'])) {
      $UCMPaymentCollectionParams['bill_city'] = $params['city-1'];
    }

    if (isset($params['state_province-1'])) {
      $UCMPaymentCollectionParams['bill_state'] = CRM_Core_PseudoConstant::stateProvinceAbbreviation(
          $params['state_province-1'] );
    }

    if (isset($params['postal_code-1'])) {
      $UCMPaymentCollectionParams['bill_zip'] = $params['postal_code-1'];
    }

    if (isset($params['email-5'])) {
      $UCMPaymentCollectionParams['bill_email'] = $params['email-5'];
    }

    // Allow further manipulation of the arguments via custom hooks ..
    CRM_Utils_Hook::alterPaymentProcessorParams($this, $params, $UCMPaymentCollectionParams);

    // Build our query string;
    $query_string = '';
    foreach ($UCMPaymentCollectionParams as $name => $value) {
      $query_string .= $name . '=' . $value . '&';
    }

    // Remove extra &
    $query_string = rtrim($query_string, '&');

    // Redirect the user to the payment url.
    CRM_Utils_System::redirect($this->_paymentProcessor['url_site'] . '?' . $query_string);

    exit();
  }
}
```


### Create Return Processing File

This is the file that is called by the payment processor after it
returns from processing the payment. Let's call it
UCMPaymentCollectionNotify.php

It should have this template.

**UCMPaymentCollectionNotify.php**

```php
<?php

session_start( );

require_once 'civicrm.config.php';
require_once 'CRM/Core/Config.php';

$config = CRM_Core_Config::singleton();

// Change this to fit your processor name.
require_once 'UCMPaymentCollectionIPN.php';

// Change this to match your payment processor class.
edu_ucmerced_payment_ucmpaymentcollection_UCMPaymentCollectionIPN::main();
```

As you can see this includes UCMPaymentCollectionIPN.php Let's create
this file now. Although this file is very large there is only a small
amount of changes needed.


**UCMPaymentCollectionIPN.php**

```php
<?php

class edu_ucmerced_payment_ucmpaymentcollection_UCMPaymentCollectionIPN extends CRM_Core_Payment_BaseIPN {

    /**
     * We only need one instance of this object. So we use the singleton
     * pattern and cache the instance in this variable
     *
     * @var object
     * @static
     */
    static private $_singleton = null;

    /**
     * mode of operation: live or test
     *
     * @var object
     * @static
     */
    static protected $_mode = null;

    static function retrieve( $name, $type, $object, $abort = true ) {
      $value = CRM_Utils_Array::value($name, $object);
      if ($abort && $value === null) {
        CRM_Core_Error::debug_log_message("Could not find an entry for $name");
        echo "Failure: Missing Parameter<p>";
        exit();
      }

      if ($value) {
        if (!CRM_Utils_Type::validate($value, $type)) {
          CRM_Core_Error::debug_log_message("Could not find a valid entry for $name");
          echo "Failure: Invalid Parameter<p>";
          exit();
        }
      }

      return $value;
    }

    /**
     * Constructor
     *
     * @param string $mode the mode of operation: live or test
     *
     * @return void
     */
    function __construct($mode, &$paymentProcessor) {
      parent::__construct();

      $this->_mode = $mode;
      $this->_paymentProcessor = $paymentProcessor;
    }

    /**
     * The function gets called when a new order takes place.
     *
     * @param xml   $dataRoot    response send by google in xml format
     * @param array $privateData contains the name value pair of <merchant-private-data>
     *
     * @return void
     *
     */
    function newOrderNotify($success, $privateData, $component, $amount, $transactionReference ) {
        $ids = $input = $params = array( );
        $input['component'] = strtolower($component);
        $ids['contact'] = self::retrieve( 'contactID', 'Integer', $privateData, true);
        $ids['contribution'] = self::retrieve( 'contributionID', 'Integer', $privateData, true);
        if ( $input['component'] == "event" ) {
            $ids['event'] = self::retrieve( 'eventID', 'Integer', $privateData, true);
            $ids['participant'] = self::retrieve( 'participantID', 'Integer', $privateData, true);
            $ids['membership'] = null;
        } else {
            $ids['membership'] = self::retrieve( 'membershipID'  , 'Integer', $privateData, false);
        }
        $ids['contributionRecur'] = $ids['contributionPage'] = null;

        if ( ! $this->validateData( $input, $ids, $objects ) ) {
            return false;
        }

        // make sure the invoice is valid and matches what we have in the contribution record
        $input['invoice'] = $privateData['invoiceID'];
        $input['newInvoice'] = $transactionReference;
        $contribution =& $objects['contribution'];
        $input['trxn_id'] = $transactionReference;

        if ( $contribution->invoice_id != $input['invoice'] ) {
          CRM_Core_Error::debug_log_message( "Invoice values dont match between database and IPN request" );
          echo "Failure: Invoice values dont match between database and IPN request<p>";
          return;
        }

        // lets replace invoice-id with Payment Processor -number because thats what is common and unique
        // in subsequent calls or notifications sent by google.
        $contribution->invoice_id = $input['newInvoice'];

        $input['amount'] = $amount;

        if ( $contribution->total_amount != $input['amount'] ) {
          CRM_Core_Error::debug_log_message( "Amount values dont match between database and IPN request" );
          echo "Failure: Amount values dont match between database and IPN request."\
                      .$contribution->total_amount."/".$input['amount']."<p>";
          return;
        }

        require_once 'CRM/Core/Transaction.php';
        $transaction = new CRM_Core_Transaction( );

        // check if contribution is already completed, if so we ignore this ipn

        if ( $contribution->contribution_status_id == 1 ) {
          CRM_Core_Error::debug_log_message( "returning since contribution has already been handled" );
          echo "Success: Contribution has already been handled<p>";
          return true;
        } else {
          /* Since trxn_id hasn't got any use here,
           * lets make use of it by passing the eventID/membershipTypeID to next level.
           * And change trxn_id to the payment processor reference before finishing db update */
          if ( $ids['event'] ) {
            $contribution->trxn_id = $ids['event'] 
              . CRM_Core_DAO::VALUE_SEPARATOR
              . $ids['participant'];
          } else {
            $contribution->trxn_id = $ids['membership'];
          }
        }
        $this->completeTransaction ( $input, $ids, $objects, $transaction);
        return true;
    }


    /**
     * singleton function used to manage this object
     *
     * @param string $mode the mode of operation: live or test
     *
     * @return object
     * @static
     */
    static function &singleton( $mode, $component, &$paymentProcessor ) {
      if ( self::$_singleton === null ) {
        self::$_singleton = new edu_ucmerced_payment_ucmpaymentcollection_UCMPaymentCollectionIPN( $mode,
                                             $paymentProcessor );
      }
      return self::$_singleton;
    }

    /**
     * The function returns the component(Event/Contribute..)and whether it is Test or not
     *
     * @param array   $privateData    contains the name-value pairs of transaction related data
     *
     * @return array context of this call (test, component, payment processor id)
     * @static
     */
    static function getContext($privateData)    {
      require_once 'CRM/Contribute/DAO/Contribution.php';

      $component = null;
      $isTest = null;

      $contributionID = $privateData['contributionID'];
      $contribution = new CRM_Contribute_DAO_Contribution();
      $contribution->id = $contributionID;

      if (!$contribution->find(true)) {
        CRM_Core_Error::debug_log_message("Could not find contribution record: $contributionID");
        echo "Failure: Could not find contribution record for $contributionID<p>";
        exit();
      }

      if (stristr($contribution->source, 'Online Contribution')) {
        $component = 'contribute';
      }
      elseif (stristr($contribution->source, 'Online Event Registration')) {
        $component = 'event';
      }
      $isTest = $contribution->is_test;

      $duplicateTransaction = 0;
      if ($contribution->contribution_status_id == 1) {
        //contribution already handled. (some processors do two notifications so this could be valid)
        $duplicateTransaction = 1;
      }

      if ($component == 'contribute') {
        if (!$contribution->contribution_page_id) {
          CRM_Core_Error::debug_log_message("Could not find contribution page for contribution record: $contributionID");
          echo "Failure: Could not find contribution page for contribution record: $contributionID<p>";
          exit();
        }

        // get the payment processor id from contribution page
        $paymentProcessorID = CRM_Core_DAO::getFieldValue('CRM_Contribute_DAO_ContributionPage',
                              $contribution->contribution_page_id, 'payment_processor_id');
      }
      else {
        $eventID = $privateData['eventID'];

        if (!$eventID) {
          CRM_Core_Error::debug_log_message("Could not find event ID");
          echo "Failure: Could not find eventID<p>";
          exit();
        }

        // we are in event mode
        // make sure event exists and is valid
        require_once 'CRM/Event/DAO/Event.php';
        $event = new CRM_Event_DAO_Event();
        $event->id = $eventID;
        if (!$event->find(true)) {
          CRM_Core_Error::debug_log_message("Could not find event: $eventID");
          echo "Failure: Could not find event: $eventID<p>";
          exit();
        }

        // get the payment processor id from contribution page
        $paymentProcessorID = $event->payment_processor_id;
      }

      if (!$paymentProcessorID) {
        CRM_Core_Error::debug_log_message("Could not find payment processor for contribution record: $contributionID");
        echo "Failure: Could not find payment processor for contribution record: $contributionID<p>";
        exit();
      }

      return array($isTest, $component, $paymentProcessorID, $duplicateTransaction);
    }

    /**
     * This method is handles the response that will be invoked (from UCMercedPaymentCollectionNotify.php) every time
     * a notification or request is sent by the UCM Payment Collection Server.
     *
     */
    static function main() {

    }
}
```

Let's start out with the minor changes that are necessary

Change the class name to fit your class. You can add any ending just as
long as it's consistent everywhere.

```php
class edu_ucmerced_payment_ucmpaymentcollection_UCMPaymentCollectionIPN extends CRM_Core_Payment_BaseIPN {
```

Again match class name to fit your class

```php
static function &singleton( $mode, $component, &$paymentProcessor ) {
if ( self::$_singleton === null ) {
            self::$_singleton = new edu_ucmerced_payment_ucmpaymentcollection_UCMPaymentCollectionIPN( $mode, $paymentProcessor );
        }
return self::$_singleton;
}
```

Insert your processing code into static function main()


**UCMPaymentCollectionIPN.php - main()**

```php
$config = CRM_Core_Config::singleton();

// Add external library to process soap transaction.
require_once('libraries/nusoap/lib/nusoap.php');

$client = new nusoap_client("https://test.example.com/verify.php", 'wsdl');
$err = $client->getError();
if ($err) {
  echo '<h2>Constructor error</h2><pre>' . $err . '</pre>';
}

// Prepare SoapHeader parameters
$param = array(
  'Username' => 'user',
  'Password' => 'password',
  'TransactionGUID' => $_GET['uuid'],
);

// This will give us some info about the transaction
$result = $client->call('PaymentVerification', array('parameters' => $param), '', '', false, true);

// Make sure there are no errors.
if (isset($result['PaymentVerificationResult']['Errors']['Error'])) {
  if ($component == "event") {
    $finalURL = CRM_Utils_System::url('civicrm/event/confirm',
         "reset=1&cc=fail&participantId={$privateData[participantID]}", false, null, false);
  } elseif ( $component == "contribute" ) {
    $finalURL = CRM_Utils_System::url('civicrm/contribute/transact',
         "_qf_Main_display=1&cancel=1&qfKey={$privateData['qfKey']}", false, null, false);
  }
}
else {
  $success = TRUE;
  $ucm_pc_values = $result['PaymentResult']['Payment'];

  $invoice_array = explode('-', $ucm_pc_values['InvoiceNumber']);

  // It is important that $privateData contains these exact keys.
  // Otherwise getContext may fail.
  $privateData['invoiceID'] = (isset($invoice_array[0])) ? $invoice_array[0] : '';
  $privateData['qfKey'] = (isset($invoice_array[1])) ? $invoice_array[1] : '';
  $privateData['contactID'] = (isset($invoice_array[2])) ? $invoice_array[2] : '';
  $privateData['contributionID'] = (isset($invoice_array[3])) ? $invoice_array[3] : '';
  $privateData['contributionTypeID'] = (isset($invoice_array[4])) ? $invoice_array[4] : '';
  $privateData['eventID'] = (isset($invoice_array[5])) ? $invoice_array[5] : '';
  $privateData['participantID'] = (isset($invoice_array[6])) ? $invoice_array[6] : '';
  $privateData['membershipID'] = (isset($invoice_array[7])) ? $invoice_array[7] : '';

  list($mode, $component, $paymentProcessorID, $duplicateTransaction) = self::getContext($privateData);
  $mode = $mode ? 'test' : 'live';

  require_once 'CRM/Core/BAO/PaymentProcessor.php';
  $paymentProcessor = CRM_Core_BAO_PaymentProcessor::getPayment($paymentProcessorID, $mode);

  $ipn=& self::singleton( $mode, $component, $paymentProcessor );

  if ($duplicateTransaction == 0) {
    // Process the transaction.
    $ipn->newOrderNotify($success, $privateData, $component,
            $ucm_pc_values['TotalAmountCharged'], $ucm_pc_values['TransactionNumber']);
  }

  // Redirect our users to the correct url.
  if ($component == "event") {
    $finalURL = CRM_Utils_System::url('civicrm/event/register',
        "_qf_ThankYou_display=1&qfKey={$privateData['qfKey']}", false, null, false);
  }
  elseif ($component == "contribute") {
    $finalURL = CRM_Utils_System::url('civicrm/contribute/transact',
        "_qf_ThankYou_display=1&qfKey={$privateData['qfKey']}", false, null, false);
  }
}

CRM_Utils_System::redirect( $finalURL );
```
### Populate Help Text on the Payment Processor Administrator Screen
To populate the blue help icons for the settings fields needed for your payment processor at **Administer -> System Settings -> Payment Processors** follow the steps below:

1. Add a template file to your extension with a `!#twig {htxt id='$ppTypeName-live-$fieldname'}` section for each settings field you are using.

    **Example:**

    The help text for the `user-name` field for a payment processor with the name 'AuthNet' would be implemented with code like this:

    ```twig
    {htxt id='AuthNet-live-user-name'}
    {ts}Generate your API Login and Transaction Key by logging in to your Merchant Account and navigating to <strong>Settings &raquo; General Security Settings</strong>.{/ts}</p>
    {/htxt}
```

    see [core /templates/CRM/Admin/Page/PaymentProcessor.hlp](https://github.com/civicrm/civicrm-core/blob/master/templates/CRM/Admin/Page/PaymentProcessor.hlp) for further examples.
1. Add that template to the `CRM_Admin_Form_PaymentProcessor` form using a buildForm hook like so:

    ```php
    if ($formName == 'CRM_Admin_Form_PaymentProcessor') {
        $templatePath = realpath(dirname(__FILE__) . "/templates");
        CRM_Core_Region::instance('form-buttons')->add(array(
          'template' => "{$templatePath}/{TEMPLATE FILE NAME}.tpl",
        ));
      }
     ```

### Add Any Additional Libraries Needed

In the case of this payment processor we needed nusoap. So in the
extension directory we created a libraries directory and put it there.
Make sure you look at any licensing restrictions before distributing
your extension with the new library.


## Testing Processor Plugins {:#testing}

Here's some suggestions of what you might test once you have written
your payment processor plug in.

!!! important
    Don't forget that you need to search specifically for TEST transactions

    ie from this page `civicrm/contribute/search&reset=1` chose "find test transactions".

### Std Payment processor tests

1. Can process Successful transaction from

    - Event
    - Contribute Form
    - Individual Contact Record (for on-site processors only)

    Transaction should show as confirmed in CiviCRM and on the payment processor

2. Can include `, . & = ' "` in address and name fields without problems. Overlong ZIP code is handled

3. Can process a failed transaction from a Contribute form

    Can fix up details & resubmit for a successful transaction
    
    e-mail address is successfully passed through to payment processor and
    payment processor sends e-mails if configured to do so.
    
    The invoice ID is processed and maintained in an adequate manner

7. Any result references and transaction codes are stored in an adequate
manner

    Recurring Payment Processor tests
    
    !!! note
        IN Paypal Manager the recurring billing profiles are in Service Settings/Recurring Billing/ Manage Profiles

1. Process a recurring contribution. Check

    - wording on confirm page is acceptable
    - wording on thankyou pages is acceptable
    - wording on any confirmation e-mails is acceptable
    - the payment processor shows the original transaction is successful 
    - the payment processor shows the correct date for the next transaction 
    - the payment processor shows the correct total number of transactions and / or the correct final date

2. Try processing different types of frequencies. Preferably test a monthly contribution on the last day of a month where there isn't a similar day in the following month (e.g. 30 January)

3. Process a transaction without filling in the total number of transactions (there should be no end date set)

4. Process a recurring contribution with the total instalments set to 1 (it should be treated as a one-off rather than a rec urring transaction). It should not show 'recurring contribution' when you search for it in CiviCRM

5. PayflowPro - check that you can edit the frequencies available on the configure contribution page form

6. Depending on your processor it may be important to identify the transactions that need to be updated or checked. You may wish to check what it being recorded in the civicrm_contribution_recur table for payment processor id, end date and next transaction date.

### Specific Live tests

1. Successful and unsuccessful REAL transactions work

2. Money vests into the bank account

3. For recurring transactions wait for the first recurrent transaction
to vest
