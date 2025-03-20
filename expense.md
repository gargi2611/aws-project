# Serverless Expense Tracker 

## Part 1: Creating a DynamoDB Table

1. **Sign in to the AWS Management Console**
   - Go to https://aws.amazon.com and click "Sign In to the Console"
   - Enter your credentials

2. **Navigate to DynamoDB**
   - In the search bar at the top, type "DynamoDB" and select it from the dropdown

3. **Create a New Table**
   - Click the "Create table" button
   - For "Table name", enter `Expenses`
   - For "Partition key", enter `expense_id` and select "String" from the dropdown
   - Scroll down to "Table settings"
   - Select "On-demand" capacity mode (this ensures you only pay for what you use)
   - Leave other settings as default
   - Click "Create table" at the bottom

4. **Wait for Table Creation**
   - This may take a minute. You'll see "Creating" status
   - When complete, the status will change to "Active"

## Part 2: Creating an AWS Lambda Function

1. **Navigate to Lambda**
   - In the AWS Console search bar, type "Lambda" and select it

2. **Create a New Lambda Function**
   - Click "Create function"
   - Select "Author from scratch"
   - For "Function name", enter `ExpenseHandler`
   - For "Runtime", select "Python 3.11" (or the latest version available)
   - Under "Permissions", expand "Change default execution role"
   - Select "Create a new role with basic Lambda permissions"
   - Click "Create function"

3. **Add DynamoDB Permissions to Your Lambda Role**
   - After the function is created, go to the "Configuration" tab
   - Click on "Permissions"
   - Click on the role name (it should look like `ExpenseHandler-role-xxxx`)
   - This will open the IAM console in a new tab
   - Click "Add permissions" > "Attach policies"
   - Search for "DynamoDB" and select "AmazonDynamoDBFullAccess"
   - Click "Add permissions" at the bottom

## Part 3: Writing Lambda Code for CRUD Operations

1. **Back in your Lambda function**:
   - Go to the "Code" tab
   - You'll see a code editor with a file named `lambda_function.py`
   - Replace the existing code with the following:

```python
import json
import boto3
import uuid
from decimal import Decimal
import os

# Initialize DynamoDB client
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Expenses')

# Custom JSON encoder to handle Decimal types
class DecimalEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, Decimal):
            return float(obj)
        return super(DecimalEncoder, self).default(obj)

def lambda_handler(event, context):
    # Print the event for debugging
    print(f"Received event: {json.dumps(event)}")
    
    # Extract HTTP method
    http_method = event.get('httpMethod')
    
    try:
        # Create a new expense
        if http_method == 'POST':
            # Parse the request body
            body = json.loads(event.get('body', '{}'))
            
            # Validate required fields
            required_fields = ['amount', 'category', 'date']
            for field in required_fields:
                if field not in body:
                    return {
                        'statusCode': 400,
                        'headers': {
                            'Content-Type': 'application/json',
                            'Access-Control-Allow-Origin': '*'
                        },
                        'body': json.dumps({'error': f'Missing required field: {field}'})
                    }
            
            # Generate a unique ID
            expense_id = str(uuid.uuid4())
            
            # Prepare the item to store
            item = {
                'expense_id': expense_id,
                'amount': Decimal(str(body['amount'])),
                'category': body['category'],
                'date': body['date'],
                'description': body.get('description', '')
            }
            
            # Save to DynamoDB
            table.put_item(Item=item)
            
            return {
                'statusCode': 201,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({
                    'message': 'Expense added successfully',
                    'expense_id': expense_id
                })
            }
        
        # Get a specific expense or list all expenses
        elif http_method == 'GET':
            # Check if we're getting a specific expense or listing all
            query_parameters = event.get('queryStringParameters', {}) or {}
            expense_id = query_parameters.get('expense_id')
            
            if expense_id:
                # Get specific expense
                response = table.get_item(Key={'expense_id': expense_id})
                item = response.get('Item')
                
                if not item:
                    return {
                        'statusCode': 404,
                        'headers': {
                            'Content-Type': 'application/json',
                            'Access-Control-Allow-Origin': '*'
                        },
                        'body': json.dumps({'error': 'Expense not found'})
                    }
                
                return {
                    'statusCode': 200,
                    'headers': {
                        'Content-Type': 'application/json',
                        'Access-Control-Allow-Origin': '*'
                    },
                    'body': json.dumps(item, cls=DecimalEncoder)
                }
            else:
                # List all expenses (simple implementation - in production consider pagination)
                response = table.scan()
                items = response.get('Items', [])
                
                return {
                    'statusCode': 200,
                    'headers': {
                        'Content-Type': 'application/json',
                        'Access-Control-Allow-Origin': '*'
                    },
                    'body': json.dumps({'expenses': items}, cls=DecimalEncoder)
                }
        
        # Update an existing expense
        elif http_method == 'PUT':
            # Parse the request body
            body = json.loads(event.get('body', '{}'))
            
            # Check if expense_id is provided
            if 'expense_id' not in body:
                return {
                    'statusCode': 400,
                    'headers': {
                        'Content-Type': 'application/json',
                        'Access-Control-Allow-Origin': '*'
                    },
                    'body': json.dumps({'error': 'expense_id is required'})
                }
            
            # Check if expense exists
            expense_id = body['expense_id']
            response = table.get_item(Key={'expense_id': expense_id})
            if 'Item' not in response:
                return {
                    'statusCode': 404,
                    'headers': {
                        'Content-Type': 'application/json',
                        'Access-Control-Allow-Origin': '*'
                    },
                    'body': json.dumps({'error': 'Expense not found'})
                }
            
            # Prepare update expression and attribute values
            update_expression = "SET "
            expression_attribute_values = {}
            
            # Handle each updatable field
            if 'amount' in body:
                update_expression += "amount=:a, "
                expression_attribute_values[':a'] = Decimal(str(body['amount']))
            
            if 'category' in body:
                update_expression += "category=:c, "
                expression_attribute_values[':c'] = body['category']
            
            if 'date' in body:
                update_expression += "date=:d, "
                expression_attribute_values[':d'] = body['date']
                
            if 'description' in body:
                update_expression += "description=:desc, "
                expression_attribute_values[':desc'] = body['description']
            
            # Remove trailing comma and space
            update_expression = update_expression[:-2]
            
            # If no fields to update
            if not expression_attribute_values:
                return {
                    'statusCode': 400,
                    'headers': {
                        'Content-Type': 'application/json',
                        'Access-Control-Allow-Origin': '*'
                    },
                    'body': json.dumps({'error': 'No fields to update'})
                }
            
            # Update the item in DynamoDB
            table.update_item(
                Key={'expense_id': expense_id},
                UpdateExpression=update_expression,
                ExpressionAttributeValues=expression_attribute_values
            )
            
            return {
                'statusCode': 200,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({'message': 'Expense updated successfully'})
            }
        
        # Delete an expense
        elif http_method == 'DELETE':
            # Get expense_id from query parameters
            query_parameters = event.get('queryStringParameters', {}) or {}
            expense_id = query_parameters.get('expense_id')
            
            if not expense_id:
                return {
                    'statusCode': 400,
                    'headers': {
                        'Content-Type': 'application/json',
                        'Access-Control-Allow-Origin': '*'
                    },
                    'body': json.dumps({'error': 'expense_id is required'})
                }
            
            # Delete the item from DynamoDB
            table.delete_item(Key={'expense_id': expense_id})
            
            return {
                'statusCode': 200,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({'message': 'Expense deleted successfully'})
            }
        
        # Handle OPTIONS requests for CORS
        elif http_method == 'OPTIONS':
            return {
                'statusCode': 200,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*',
                    'Access-Control-Allow-Headers': 'Content-Type,X-Amz-Date,Authorization,X-Api-Key',
                    'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS'
                },
                'body': json.dumps({})
            }
        
        # Method not allowed
        else:
            return {
                'statusCode': 405,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({'error': 'Method not allowed'})
            }

    except Exception as e:
        # Handle any errors
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({'error': str(e)})
        }

```

2. **Deploy the Code**:
   - Click the "Deploy" button to save and deploy your code
   - Wait for the "Function successfully updated" message

3. **Test the Function**:
   - Click the "Test" tab at the top
   - Click "Create new event"
   - Name the event "CreateExpenseTest"
   - Replace the default JSON with this sample for testing a POST request:

```json
{
  "httpMethod": "POST",
  "body": "{\"amount\": 50.25, \"category\": \"Food\", \"date\": \"2025-03-20\", \"description\": \"Lunch\"}"
}
```

   - Click "Save" and then "Test"
   - You should see a success message with a generated expense_id

## Part 4: Creating an API Gateway

1. **Navigate to API Gateway**
   - In the AWS Console search bar, type "API Gateway" and select it

2. **Create a New API**
   - Click "Create API"
   - Under "HTTP API", click "Build"
   - In the "Integrations" section, select "Lambda"
   - Select your region and then in the Lambda function dropdown, select "ExpenseHandler"
   - In the "API name" field at the top, enter "ExpenseTrackerAPI"
   - Click "Next"

3. **Configure Routes**
   - You'll need to create four routes for CRUD operations. Click "Add route" for each:
     
     a. First route:
     - Method: POST
     - Resource path: /expense
     - Integration target: ExpenseHandler
     
     b. Second route:
     - Method: GET
     - Resource path: /expense
     - Integration target: ExpenseHandler
     
     c. Third route:
     - Method: PUT
     - Resource path: /expense
     - Integration target: ExpenseHandler
     
     d. Fourth route:
     - Method: DELETE
     - Resource path: /expense
     - Integration target: ExpenseHandler
     
     e. Fifth route (for CORS):
     - Method: OPTIONS
     - Resource path: /expense
     - Integration target: ExpenseHandler
   
   - Click "Next"

4. **Configure Stage**
   - For "Stage name", enter "$default" (this creates a default stage)
   - Enable Auto-deploy
   - Click "Next"

5. **Review and Create**
   - Review all settings
   - Click "Create" to create your API

6. **Note Your API URL**
   - After creation, you'll see a screen with details about your API
   - Note the "Invoke URL" - this is the base URL for your API endpoints

## Part 5: Testing Your API

You can test your API using tools like Postman, cURL, or even your web browser (for GET requests). Here are commands for testing each endpoint:

1. **Create an Expense (POST)**:
```bash
curl -X POST \
  https://your-api-id.execute-api.your-region.amazonaws.com/expense \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50.25,
    "category": "Food",
    "date": "2025-03-20",
    "description": "Lunch"
  }'
```

2. **Get All Expenses (GET)**:
```bash
curl -X GET \
  https://your-api-id.execute-api.your-region.amazonaws.com/expense
```

3. **Get a Specific Expense (GET with query parameter)**:
```bash
curl -X GET \
  "https://your-api-id.execute-api.your-region.amazonaws.com/expense?expense_id=YOUR_EXPENSE_ID"
```

4. **Update an Expense (PUT)**:
```bash
curl -X PUT \
  https://your-api-id.execute-api.your-region.amazonaws.com/expense \
  -H "Content-Type: application/json" \
  -d '{
    "expense_id": "YOUR_EXPENSE_ID",
    "amount": 60.50,
    "category": "Dining"
  }'
```

5. **Delete an Expense (DELETE)**:
```bash
curl -X DELETE \
  "https://your-api-id.execute-api.your-region.amazonaws.com/expense?expense_id=YOUR_EXPENSE_ID"
```

Replace `your-api-id.execute-api.your-region.amazonaws.com` with your actual API URL and `YOUR_EXPENSE_ID` with an actual expense ID created earlier.

## Part 6: Building a Simple Frontend (Optional)

If you want to add a frontend, you can use simple HTML, CSS, and JavaScript. Here's a basic example:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Expense Tracker</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
        }
        .form-group {
            margin-bottom: 15px;
        }
        label {
            display: block;
            margin-bottom: 5px;
        }
        input, select {
            width: 100%;
            padding: 8px;
            box-sizing: border-box;
        }
        button {
            background-color: #4CAF50;
            color: white;
            padding: 10px 15px;
            border: none;
            cursor: pointer;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        th, td {
            border: 1px solid #ddd;
            padding: 8px;
            text-align: left;
        }
        th {
            background-color: #f2f2f2;
        }
    </style>
</head>
<body>
    <h1>Expense Tracker</h1>
    
    <div id="expense-form">
        <h2>Add New Expense</h2>
        <form id="addExpenseForm">
            <div class="form-group">
                <label for="amount">Amount ($):</label>
                <input type="number" id="amount" step="0.01" required>
            </div>
            <div class="form-group">
                <label for="category">Category:</label>
                <select id="category" required>
                    <option value="">Select a category</option>
                    <option value="Food">Food</option>
                    <option value="Transportation">Transportation</option>
                    <option value="Entertainment">Entertainment</option>
                    <option value="Housing">Housing</option>
                    <option value="Utilities">Utilities</option>
                    <option value="Healthcare">Healthcare</option>
                    <option value="Other">Other</option>
                </select>
            </div>
            <div class="form-group">
                <label for="date">Date:</label>
                <input type="date" id="date" required>
            </div>
            <div class="form-group">
                <label for="description">Description:</label>
                <input type="text" id="description">
            </div>
            <button type="submit">Add Expense</button>
        </form>
    </div>
    
    <div id="expenses-list">
        <h2>Expenses</h2>
        <table id="expensesTable">
            <thead>
                <tr>
                    <th>Date</th>
                    <th>Category</th>
                    <th>Description</th>
                    <th>Amount</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody id="expensesTableBody">
                <!-- Expenses will be added here -->
            </tbody>
        </table>
    </div>

    <script>
        // Replace with your actual API Gateway URL
        const API_URL = 'https://your-api-id.execute-api.your-region.amazonaws.com/expense';
        
        // Load expenses when page loads
        document.addEventListener('DOMContentLoaded', loadExpenses);
        
        // Add form submission handler
        document.getElementById('addExpenseForm').addEventListener('submit', addExpense);
        
        // Function to load all expenses
        async function loadExpenses() {
            try {
                const response = await fetch(API_URL);
                if (!response.ok) {
                    throw new Error('Failed to fetch expenses');
                }
                
                const data = await response.json();
                const expenses = data.expenses || [];
                
                const tableBody = document.getElementById('expensesTableBody');
                tableBody.innerHTML = ''; // Clear existing rows
                
                expenses.forEach(expense => {
                    const row = document.createElement('tr');
                    
                    // Format date
                    const dateCell = document.createElement('td');
                    dateCell.textContent = expense.date;
                    row.appendChild(dateCell);
                    
                    // Category
                    const categoryCell = document.createElement('td');
                    categoryCell.textContent = expense.category;
                    row.appendChild(categoryCell);
                    
                    // Description
                    const descriptionCell = document.createElement('td');
                    descriptionCell.textContent = expense.description || '';
                    row.appendChild(descriptionCell);
                    
                    // Amount
                    const amountCell = document.createElement('td');
                    amountCell.textContent = `$${parseFloat(expense.amount).toFixed(2)}`;
                    row.appendChild(amountCell);
                    
                    // Actions
                    const actionsCell = document.createElement('td');
                    
                    // Delete button
                    const deleteButton = document.createElement('button');
                    deleteButton.textContent = 'Delete';
                    deleteButton.style.backgroundColor = '#f44336';
                    deleteButton.style.marginRight = '5px';
                    deleteButton.addEventListener('click', () => deleteExpense(expense.expense_id));
                    actionsCell.appendChild(deleteButton);
                    
                    row.appendChild(actionsCell);
                    
                    tableBody.appendChild(row);
                });
                
            } catch (error) {
                console.error('Error loading expenses:', error);
                alert('Failed to load expenses. See console for details.');
            }
        }
        
        // Function to add a new expense
        async function addExpense(event) {
            event.preventDefault();
            
            const amount = document.getElementById('amount').value;
            const category = document.getElementById('category').value;
            const date = document.getElementById('date').value;
            const description = document.getElementById('description').value;
            
            const expenseData = {
                amount: parseFloat(amount),
                category,
                date,
                description
            };
            
            try {
                const response = await fetch(API_URL, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify(expenseData)
                });
                
                if (!response.ok) {
                    throw new Error('Failed to add expense');
                }
                
                // Reset form
                document.getElementById('addExpenseForm').reset();
                
                // Reload expenses
                loadExpenses();
                
            } catch (error) {
                console.error('Error adding expense:', error);
                alert('Failed to add expense. See console for details.');
            }
        }
        
        // Function to delete an expense
        async function deleteExpense(expenseId) {
            if (!confirm('Are you sure you want to delete this expense?')) {
                return;
            }
            
            try {
                const response = await fetch(`${API_URL}?expense_id=${expenseId}`, {
                    method: 'DELETE'
                });
                
                if (!response.ok) {
                    throw new Error('Failed to delete expense');
                }
                
                // Reload expenses
                loadExpenses();
                
            } catch (error) {
                console.error('Error deleting expense:', error);
                alert('Failed to delete expense. See console for details.');
            }
        }
    </script>
</body>
</html>

```

To use this frontend:
1. Save the HTML to a file (e.g., `index.html`)
2. Update the `API_URL` variable with your actual API Gateway URL
3. Open the file in your web browser

For a production application, you would typically host this on:
- Amazon S3 (static website hosting)
- AWS Amplify
- Amazon CloudFront (for global distribution)

## Part 7: Monitoring and Optimization

1. **Set Up CloudWatch Monitoring**
   - In the Lambda console, go to the "Monitor" tab
   - View the CloudWatch metrics for invocations, errors, and duration
   - Click "View logs in CloudWatch" to see detailed logs

2. **Optimize DynamoDB**
   - Consider adding a Global Secondary Index (GSI) if you need to query by category or date:
     - Go to the DynamoDB console > Select your Expenses table
     - Go to the "Indexes" tab > "Create index"
     - For "Partition key", choose the field you want to query by (e.g., "category")
     - Name the index (e.g., "CategoryIndex")
     - Click "Create index"

3. **Optimize Lambda Performance**
   - In the Lambda console, go to the "Configuration" tab > "General configuration"
   - Click "Edit"
   - Adjust the memory allocation (higher memory = more CPU = faster execution)
   - Adjust timeout if needed (default is 3 seconds, which may be too short)
   - Click "Save"

## Common Issues and Troubleshooting

1. **CORS Errors**
   - If you get CORS errors when accessing the API from a browser, verify that your Lambda function returns the proper CORS headers
   - Ensure your API Gateway has OPTIONS method configured

2. **Lambda Permissions**
   - If you get access denied errors, check that your Lambda execution role has the right permissions for DynamoDB

3. **API Gateway Integration**
   - If API calls fail, check the API Gateway integration settings
   - Verify the routes are correctly mapped to your Lambda function

4. **DynamoDB Capacity**
   - If you experience throttling, consider adjusting your capacity mode

## Next Steps to Enhance Your Application

1. **Add User Authentication**
   - Implement Amazon Cognito for user sign-up/sign-in
   - Secure the API endpoints with Cognito authorizers

2. **Add Data Visualization**
   - Use Chart.js or D3.js in your frontend to visualize spending patterns
   - Consider integrating with Amazon QuickSight for advanced analytics

3. **Deploy with Infrastructure as Code**
   - Use AWS SAM (Serverless Application Model) or AWS CDK to deploy your application

4. **Add Budget Alerts**
   - Implement a feature to alert users when they exceed budget thresholds
