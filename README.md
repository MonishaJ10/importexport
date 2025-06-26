{
  "name": "ForeignExchangeRecon",
  "description": "Auto import of reconciliation model",
  "serviceName": "INTMATCH_3",
  "type": "RECONCILIATION",
  "fields": [
    {
      "name": "TransactionId",
      "type": "STRING",
      "isKey": true
    },
    {
      "name": "Amount",
      "type": "DECIMAL",
      "aggregation": "SUM"
    },
    {
      "name": "Currency",
      "type": "STRING"
    },
    {
      "name": "TradeDate",
      "type": "DATE"
    }
  ],
  "matchRules": [
    {
      "name": "MatchByTxnID",
      "priority": 1,
      "matchConditions": [
        {
          "leftField": "TransactionId",
          "operator": "EQUALS",
          "rightField": "TransactionId"
        }
      ]
    }
  ]
}







{
  "modelName": "ForeignExchangeModel",
  "description": "Sample Reconciliation Model for Foreign Exchange",
  "version": "1.0",
  "reconService": "INTMATCH_3",
  "mode": "Auto",
  "frequency": "OnDemand",
  "fields": [
    {
      "fieldName": "TransactionId",
      "fieldType": "string",
      "isKey": true
    },
    {
      "fieldName": "Amount",
      "fieldType": "decimal",
      "aggregation": "sum"
    },
    {
      "fieldName": "Currency",
      "fieldType": "string"
    },
    {
      "fieldName": "TradeDate",
      "fieldType": "date"
    }
  ],
  "matchRules": [
    {
      "ruleName": "MatchByTransaction",
      "criteria": [
        {
          "sourceField": "TransactionId",
          "targetField": "TransactionId",
          "comparison": "equals"
        }
      ]
    }
  ]
}
