# step-funciotn

{
  "Comment": "A description of my state machine",
  "StartAt": "Inicializar contexto",
  "States": {
    "Inicializar contexto": {
      "Type": "Pass",
      "Next": "Aguardar chegada de mais arquivos",
      "Assign": {
        "jobRunId": "{% $states.input.id %}",
        "tipoCarga": "{% $split($states.input.detail.object.key, '/')[1] %}"
      }
    },
    "Aguardar chegada de mais arquivos": {
      "Type": "Wait",
      "Seconds": 10,
      "Next": "Movimentar Arquivos"
    },
    "Movimentar Arquivos": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Output": {
        "pathParaProcessar": "{% $states.result.Payload.body.path_a_processar %}"
      },
      "Arguments": {
        "FunctionName": "arn:aws:lambda:us-east-1:xxxxxxxxx:function:lambdamovimentaarquivos",
        "Payload": {
          "bucketOrigem": "{% $states.input.detail.bucket.name  %}",
          "pathOrigem": "{% $split($states.input.detail.object.key, '/')[0] & '/' & $split($states.input.detail.object.key, '/')[1]  %}",
          "bucketDestino": "{% $states.input.detail.bucket.name  %}",
          "pathDestino": "{% 'PROCESSANDO' & '/' & $split($states.input.detail.object.key, '/')[0] & '/' & $split($states.input.detail.object.key, '/')[1] & '/' & $split($states.input.time, 'T')[0] & '/' & $states.input.id %}"
        }
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException",
            "Lambda.TooManyRequestsException"
          ],
          "IntervalSeconds": 1,
          "MaxAttempts": 3,
          "BackoffRate": 2,
          "JitterStrategy": "FULL"
        }
      ],
      "End": true
    }
  },
  "QueryLanguage": "JSONata"
}
