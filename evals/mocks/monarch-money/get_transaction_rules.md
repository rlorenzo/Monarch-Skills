[
 {
  "id": "rule-1",
  "order": 1,
  "merchant_criteria": [
   {
    "operator": "contains",
    "value": "HARBOR CARD AUTOPAY"
   }
  ],
  "merchant_name_criteria": null,
  "original_statement_criteria": null,
  "amount_criteria": {
   "operator": "gt",
   "isExpense": false,
   "value": 0
  },
  "category_ids": null,
  "account_ids": [
   "acc-card"
  ],
  "use_original_statement": true,
  "set_category_action": {
   "id": "cat-paychecks",
   "name": "Paychecks"
  },
  "set_merchant_action": null,
  "add_tags_action": null,
  "link_goal_action": null,
  "hide_from_reports_action": null
 }
]
