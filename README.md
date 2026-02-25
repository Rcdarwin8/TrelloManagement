# TrelloManagement
performing management automation on trello using api automation.

# Run Using newman
newman run Trello_Automation.postman_collection.json \
-e Trello_Env.json \
-r htmlextra \
--reporter-htmlextra-title "Trello API Automation Report" \
--reporter-htmlextra-export ./report.html

