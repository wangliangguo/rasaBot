## say goodbye
* goodbye
  - utter_goodbye

## Some question from FAQ
* faq
 - respond_faq


## contact form
* contact_human
    - contact_form                   <!--Run the sales_form action-->
    - form{"name": "contact_form"}   <!--Activate the form-->
    - form{"name": null}           <!--Deactivate the form-->

## just sales, continue
* contact_human
    - contact_form
    - form{"name": "contact_form"}
* faq
    - respond_faq
    - contact_form
    - form{"name": null}


## explain name
* contact_human
    - contact_form
    - form{"name": "contact_form"}
    - slot{"requested_slot": "name"}
* explain
    - utter_explain_why_name
    - contact_form
    - form{"name": null}

## explain phone
* contact_human
    - contact_form
    - form{"name": "contact_form"}
    - slot{"requested_slot": "phone"}
* explain
    - utter_explain_why_phone
    - contact_form
    - form{"name": null}

## out of scope
* out_of_scope
  - utter_out_of_scope
