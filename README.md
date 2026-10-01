{
  "name": "GitHub Auto Triager & Duplicate Detector",
  "nodes": [
    {
      "parameters": {
        "authentication": "oAuth2",
        "owner": {
          "__rl": true,
          "value": "anjali4796",
          "mode": "name"
        },
        "repository": {
          "__rl": true,
          "value": "Anjali",
          "mode": "name"
        },
        "events": [
          "issues"
        ],
        "options": {}
      },
      "id": "dc79667b-30f8-4e2d-a296-d8a4e8b05eae",
      "name": "Github Trigger",
      "type": "n8n-nodes-base.githubTrigger",
      "typeVersion": 1,
      "position": [
        -112,
        128
      ],
      "webhookId": "bb02020c-da77-49d0-a138-ecf1068713a5",
      "credentials": {
        "githubOAuth2Api": {
          "id": "j5uRprLm16v0gnnr",
          "name": "GitHub account"
        }
      }
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "leftValue": "={{ $json.body.action }}",
              "rightValue": "opened",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "fa963c91-58b9-42ae-9944-62ce31ad5141",
      "name": "Only newly opened issues",
      "type": "n8n-nodes-base.filter",
      "typeVersion": 2.2,
      "position": [
        208,
        128
      ]
    },
    {
      "parameters": {
        "authentication": "oAuth2",
        "resource": "repository",
        "owner": {
          "__rl": true,
          "value": "anjali4796",
          "mode": "name"
        },
        "repository": {
          "__rl": true,
          "value": "Anjali",
          "mode": "name"
        },
        "returnAll": true,
        "getRepositoryIssuesFilters": {
          "state": "all"
        }
      },
      "id": "12e496ce-6bf4-4382-ad66-20f2c6aab166",
      "name": "Get existing issues",
      "type": "n8n-nodes-base.github",
      "typeVersion": 1.1,
      "position": [
        464,
        128
      ],
      "executeOnce": true,
      "alwaysOutputData": true,
      "webhookId": "5c3009d2-8706-41ef-bb91-b7c280f1c403",
      "credentials": {
        "githubOAuth2Api": {
          "id": "j5uRprLm16v0gnnr",
          "name": "GitHub account"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "// New issue comes from the trigger; existing issues come from the input\nconst newIssue = $('Github Trigger').first().json.body.issue;\nconst text = ((newIssue.title || '') + ' ' + (newIssue.body || '')).toLowerCase();\n// Whole-word match (allowing common endings) so 'down' does not match 'dropdown' and 'doc' does not match 'docker'\nconst has = (w) => w === '?' ? text.includes('?') : new RegExp('(^|[^a-z0-9])' + w + '(s|es|d|ed|ing)?($|[^a-z0-9])').test(text);\n\n// ---------- 1. Auto-triage: type label ----------\nconst typeRules = [\n  { label: 'bug', words: ['bug', 'error', 'crash', 'broken', 'not working', 'fail', 'failure', 'exception', 'issue with', 'doesn\\'t work', 'wrong'] },\n  { label: 'enhancement', words: ['feature', 'add', 'request', 'improve', 'improvement', 'enhancement', 'support for', 'would be nice', 'suggest'] },\n  { label: 'documentation', words: ['doc', 'readme', 'typo', 'documentation', 'guide', 'tutorial'] },\n  { label: 'question', words: ['how to', 'how do', 'question', 'help', 'why', '?'] }\n];\nlet typeLabel = 'needs-triage';\nfor (const rule of typeRules) {\n  if (rule.words.some(has)) { typeLabel = rule.label; break; }\n}\n\n// ---------- 2. Auto-triage: priority label ----------\nlet priority = 'priority: low';\nif (['urgent', 'critical', 'crash', 'security', 'data loss', 'production', 'down', 'asap', 'blocker'].some(has)) {\n  priority = 'priority: high';\n} else if (typeLabel === 'bug' || ['important', 'soon', 'major'].some(has)) {\n  priority = 'priority: medium';\n}\n\n// ---------- 3. Duplicate detection ----------\nconst normalize = (t) => (t || '').toLowerCase().replace(/[^a-z0-9 ]/g, ' ').split(/\\s+/).filter(w => w.length > 2);\nconst newWords = new Set(normalize(newIssue.title));\nlet best = null;\nlet bestScore = 0;\nfor (const item of $input.all()) {\n  const old = item.json;\n  if (!old.number || old.number === newIssue.number) continue; // skip the new issue itself / empty list\n  if (old.pull_request) continue; // skip pull requests\n  const oldWords = new Set(normalize(old.title));\n  if (newWords.size === 0 || oldWords.size === 0) continue;\n  const common = [...newWords].filter(w => oldWords.has(w)).length;\n  const score = common / new Set([...newWords, ...oldWords]).size;\n  if (score > bestScore) { bestScore = score; best = old; }\n}\nconst isDuplicate = !!best && bestScore >= 0.5;\n\nreturn [{ json: {\n  newNumber: newIssue.number,\n  newTitle: newIssue.title,\n  typeLabel,\n  priority,\n  isDuplicate,\n  duplicateOf: isDuplicate ? best.number : null,\n  duplicateTitle: isDuplicate ? best.title : null,\n  similarity: Math.round(bestScore * 100)\n} }];"
      },
      "id": "e5e4b74a-3683-4334-95cf-bfbf33c9f1fa",
      "name": "Triage and find duplicate",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        752,
        128
      ]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.github.com/repos/anjali4796/Anjali/issues/{{ $json.newNumber }}/labels",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "githubOAuth2Api",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ labels: $json.isDuplicate ? [$json.typeLabel, $json.priority, 'duplicate'] : [$json.typeLabel, $json.priority] }) }}",
        "options": {}
      },
      "id": "da296976-d125-41df-91ff-86aa7d666d3f",
      "name": "Apply triage labels",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        1024,
        128
      ],
      "credentials": {
        "githubOAuth2Api": {
          "id": "j5uRprLm16v0gnnr",
          "name": "GitHub account"
        }
      }
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 3
          },
          "conditions": [
            {
              "leftValue": "={{ $('Triage and find duplicate').item.json.isDuplicate }}",
              "rightValue": "",
              "operator": {
                "type": "boolean",
                "operation": "true",
                "singleValue": true
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "9dad2d91-42b9-461e-88a9-b5f690ba3442",
      "name": "Is duplicate?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.3,
      "position": [
        1024,
        320
      ]
    },
    {
      "parameters": {
        "authentication": "oAuth2",
        "operation": "createComment",
        "owner": {
          "__rl": true,
          "value": "anjali4796",
          "mode": "name"
        },
        "repository": {
          "__rl": true,
          "value": "Anjali",
          "mode": "name"
        },
        "issueNumber": "={{ $('Triage and find duplicate').item.json.newNumber }}",
        "body": "=Possible duplicate found!\nThis issue looks similar to #{{ $('Triage and find duplicate').item.json.duplicateOf }} - \"{{ $('Triage and find duplicate').item.json.duplicateTitle }}\" ({{ $('Triage and find duplicate').item.json.similarity }}% title match).\nLabels added: {{ $('Triage and find duplicate').item.json.typeLabel }}, {{ $('Triage and find duplicate').item.json.priority }}, duplicate.\nPlease check if it's the same issue."
      },
      "id": "e83e8949-dac3-45dd-a718-c5a1bd48a690",
      "name": "Comment possible duplicate",
      "type": "n8n-nodes-base.github",
      "typeVersion": 1.1,
      "position": [
        1248,
        224
      ],
      "webhookId": "2ae46084-1e80-40e1-acbd-345e5e2fa659",
      "credentials": {
        "githubOAuth2Api": {
          "id": "j5uRprLm16v0gnnr",
          "name": "GitHub account"
        }
      }
    },
    {
      "parameters": {
        "authentication": "oAuth2",
        "operation": "createComment",
        "owner": {
          "__rl": true,
          "value": "anjali4796",
          "mode": "name"
        },
        "repository": {
          "__rl": true,
          "value": "Anjali",
          "mode": "name"
        },
        "issueNumber": "={{ $('Triage and find duplicate').item.json.newNumber }}",
        "body": "=Thanks for opening this issue! It was automatically triaged.\nType: {{ $('Triage and find duplicate').item.json.typeLabel }}\nPriority: {{ $('Triage and find duplicate').item.json.priority }}\nNo similar existing issue was found."
      },
      "id": "cc00d8e3-3543-48c0-9563-0f2c7c8504f4",
      "name": "Comment triage result",
      "type": "n8n-nodes-base.github",
      "typeVersion": 1.1,
      "position": [
        1248,
        416
      ],
      "webhookId": "afcac72a-6cf2-436a-9fe2-def6c4beb1ef",
      "credentials": {
        "githubOAuth2Api": {
          "id": "j5uRprLm16v0gnnr",
          "name": "GitHub account"
        }
      }
    }
  ],
  "pinData": {},
  "connections": {
    "Github Trigger": {
      "main": [
        [
          {
            "node": "Only newly opened issues",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Only newly opened issues": {
      "main": [
        [
          {
            "node": "Get existing issues",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get existing issues": {
      "main": [
        [
          {
            "node": "Triage and find duplicate",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Triage and find duplicate": {
      "main": [
        [
          {
            "node": "Apply triage labels",
            "type": "main",
            "index": 0
          },
          {
            "node": "Is duplicate?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Is duplicate?": {
      "main": [
        [
          {
            "node": "Comment possible duplicate",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Comment triage result",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1",
    "binaryMode": "separate"
  },
  "versionId": "4bd7b04e-c6c9-4169-8528-695cf01cb4f7",
  "meta": {
    "instanceId": "32e13b03b385f131e79c9995096a3b774322ef463efaafa8ec44993816588bb5"
  },
  "nodeGroups": [],
  "id": "XQk02WCiGArdWMqQ",
  "tags": []
}
