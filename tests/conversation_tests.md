#### This file contains tests to evaluate that your bot behaves as expected.
#### If you want to learn more, please see the docs: https://rasa.com/docs/rasa/user-guide/testing-your-assistant/

## happy path 1
* greet: hello there!
  - utter_greet


## happy path 2
* greet: hello there!
  - utter_greet
* goodbye: bye-bye!
  - utter_goodbye


## say goodbye
* goodbye: bye-bye!
  - utter_goodbye
