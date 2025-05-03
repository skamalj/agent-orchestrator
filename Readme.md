
# Agent Flow State Machine with AWS Step function

This AWS Step Functions state machine orchestrates a flow of agent interactions by dynamically invoking Lambda functions based on agent configurations stored in DynamoDB.

## Overview

The state machine implements a chain of agent interactions where each agent can process a message and determine the next agent in the flow. The flow continues until an agent designates 'END' as the next agent.

## Components

### DynamoDB Table

- **Table Name**: `AgentRegistry`
- **Schema**:
  - Primary Key: `agent_name` (String)
  - Attributes:
    - `lambda_arn` (String) - ARN of the Lambda function associated with the agent

## Input Format

The state machine expects an initial input in the following format:

```
{
    "nextagent": "string",      // Name of the first agent to invoke
    "fromagent": "string",     // Name of the agent sending the message
    "message": "any",         // Message payload
    "thread_id": "string",    // Unique identifier for the conversation thread
    "channel_type": "string", // Communication channel type
    "from": "string"          // Original sender identifier
}
```

## Lambda Function Requirements

Each Lambda function in the agent chain must:

1. Accept input in the format:
```
{
    "taskToken": "string",    // Step Functions task token
    "input": {                // Original input payload
        "nextagent": "string",
        "fromagent": "string",
        "message": "any",
        "thread_id": "string",
        "channel_type": "string",
        "from": "string"
    }
}
```

2. Return output in the format:
```
{
    "fromagent": "string",     // Current agent name
    "nextagent": "string",     // Next agent to invoke (or 'END')
    "message": "any",         // Processed/modified message
    "thread_id": "string",    // Thread identifier
    "channel_type": "string", // Channel type
    "from": "string"          // Original sender
}
```

## Required Environment/Resources

1. AWS Step Functions service role with permissions:
   - `dynamodb:GetItem` on the AgentRegistry table
   - `lambda:InvokeFunction` on all agent Lambda functions

2. DynamoDB table `AgentRegistry` must be created and populated with agent configurations

3. All referenced Lambda functions must be deployed and their ARNs registered in the AgentRegistry table

## Flow Logic

1. Checks if the next agent is 'END'
2. Looks up the Lambda ARN for the next agent from DynamoDB
3. Prepares and invokes the Lambda function
4. Waits for the Lambda to complete using task tokens
5. Processes the result and continues the chain

## Error Handling

The state machine will fail if:
- The specified agent is not found in the AgentRegistry table
- The Lambda function fails to execute
- The Lambda function times out
- The response format is invalid

## Best Practices

1. Implement proper error handling in Lambda functions
2. Use appropriate Lambda timeout values
3. Monitor state machine executions using CloudWatch
4. Implement dead-letter queues for failed executions