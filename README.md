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
        "tipoCarga": "{% $split($states.input.detail.object.key, '/')[1] %}",
        "custodia": "{% $split($states.input.detail.object.key, '/')[0] %}"
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
        "pathParaProcessar": "{% $states.result.Payload.body.path_a_processar %}",
        "dominiosParaProcessar": [
          "OPERACAO",
          "ORIGINACAO",
          "PARCELA"
        ]
      },
      "Arguments": {
        "FunctionName": "arn:aws:lambda:us-east-1:xxxxxx:function:lambdamovimentaarquivos",
        "Payload": {
          "bucketOrigem": "{% $states.input.detail.bucket.name %}",
          "pathOrigem": "{% $split($states.input.detail.object.key, '/')[0] & '/' & $split($states.input.detail.object.key, '/')[1] %}",
          "bucketDestino": "{% $states.input.detail.bucket.name %}",
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
      "Next": "Iterar sobre domínios"
    },
    "Iterar sobre domínios": {
      "Type": "Map",
      "ItemProcessor": {
        "ProcessorConfig": {
          "Mode": "INLINE"
        },
        "StartAt": "Transformar Dados",
        "States": {
          "Transformar Dados": {
            "Type": "Task",
            "Resource": "arn:aws:states:::glue:startJobRun.sync",
            "Arguments": {
              "JobName": "transformardados",
              "Arguments": {
                "--DOMINIO": "{% $states.input %}",
                "--CUSTODIA": "{% $custodia %}",
                "--TIPO_CARGA": "{% $tipoCarga %}"
              }
            },
            "End": true
          }
        }
      },
      "End": true,
      "Items": "{% $states.input.dominiosParaProcessar %}"
    }
  },
  "QueryLanguage": "JSONata"
}
