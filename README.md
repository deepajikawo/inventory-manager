# Product Inventory Application
## A simple inventory application using Google form
    to input data and use google sheet to store the 
    inventory data

## Google Docs + Google Apps Script
    Automate document generation using Google Apps Script.
Use Google Forms + Google Docs to auto-generate contracts, invoices, and agreements.
Free with a Google account.


Explain the use of Google Forms + Google Docs to auto-generate contracts, invoices, and agreements.
Reasoned for a couple of seconds

Using Google Forms and Google Docs together allows you to streamline the creation of contracts, invoices, and agreements by automating the data collection and document generation process. Here’s how it works:

Data Collection with Google Forms:

Setup: Create a Google Form with fields corresponding to the information needed in your document (e.g., client name, address, service details, pricing, dates, and other terms).
User Input: When someone fills out the form, their responses are automatically recorded in a connected Google Sheet.
Template Creation in Google Docs:

Design a Template: Draft a Google Docs template that serves as your contract, invoice, or agreement. In the template, insert placeholders (like {{client_name}}, {{date}}, or {{amount}}) where you want the form responses to appear.
Customization: This template can include your branding, standard terms, and formatting that will be consistent across all generated documents.
Automation with Google Apps Script:

Script Development: Write a Google Apps Script that triggers when a new form submission is received.
Data Merging: The script retrieves the form data from the Google Sheet, makes a copy of the Google Docs template, and replaces the placeholders with the actual data from the submission.
Output Options: After generating the document, the script can save it in Google Drive, convert it to PDF, and even automatically email the document to relevant parties.
Putting It All Together:

Workflow: Once set up, your process will work as follows: a user fills out the Google Form → the response is saved in a Google Sheet → the Apps Script automatically creates a personalized document from the template → the final document is stored or shared as needed.

Benefits: This approach saves time by eliminating manual data entry and ensures consistency across all documents while leveraging free tools available in the Google ecosystem.
This solution is especially useful for small businesses or individual professionals who need to generate standardized documents quickly without investing in expensive software.

### code
    code
``` javascript
   
function processTransactionAndUpdateInventory() {
  var firebaseUrl = "YOUR_FIREBASE_DATABASE_URL"; // Firebase Realtime Database URL
  var firebaseSecret = "YOUR_FIREBASE_SECRET_KEY"; // Firebase Database secret
  var adminEmail = "admin@example.com"; // Replace with admin email for alerts
  var sheet = SpreadsheetApp.getActiveSpreadsheet();
  
  var transactionsSheet = sheet.getSheetByName("Transactions");
  var inventorySheet = sheet.getSheetByName("Inventory");
  
  var lastRow = transactionsSheet.getLastRow();
  var transactionData = transactionsSheet.getRange(lastRow, 1, 1, transactionsSheet.getLastColumn()).getValues()[0];

  var transactionDate = new Date(transactionData[0]);
  var productName = transactionData[1];
  var quantitySold = parseInt(transactionData[2]);
  var customerName = transactionData[3];
  var pricePerUnit = parseFloat(transactionData[4]);
  var totalPrice = parseFloat(transactionData[5]);

  // Find the product in the inventory sheet
  var inventoryData = inventorySheet.getDataRange().getValues();
  var inventoryRow = -1;
  
  for (var i = 1; i < inventoryData.length; i++) {
    if (inventoryData[i][0] === productName) {
      inventoryRow = i;
      break;
    }
  }

  if (inventoryRow === -1) {
    Logger.log("Product not found in inventory.");
    return;
  }

  // Update inventory stock
  var currentStock = parseInt(inventoryData[inventoryRow][1]);
  var newStock = currentStock - quantitySold;

  if (newStock < 0) {
    Logger.log("Not enough stock for " + productName);
    return;
  }

  inventorySheet.getRange(inventoryRow + 1, 2).setValue(newStock); // Update stock in sheet

  // Send Low Stock Alert
  if (newStock < 5) {
    var emailBody = `
      <h3>Low Stock Alert</h3>
      <p>Product: <strong>${productName}</strong></p>
      <p>Current Stock: <strong>${newStock}</strong></p>
      <p>Please restock soon!</p>
    `;
    MailApp.sendEmail({
      to: adminEmail,
      subject: "⚠️ Low Stock Alert: " + productName,
      htmlBody: emailBody
    });

    // Save Low Stock Alert to Firebase
    var lowStockData = {
      productName: productName,
      stockQuantity: newStock,
      alert: "Low Stock! Please restock."
    };

    var lowStockOptions = {
      method: "post",
      contentType: "application/json",
      payload: JSON.stringify(lowStockData)
    };

    UrlFetchApp.fetch(firebaseUrl + "/low_stock_alerts.json?auth=" + firebaseSecret, lowStockOptions);
  }

  // Save Transaction to Firebase
  var transactionRecord = {
    date: transactionDate.toISOString(),
    productName: productName,
    quantitySold: quantitySold,
    customerName: customerName,
    pricePerUnit: pricePerUnit,
    totalPrice: totalPrice
  };

  var inventoryUpdate = {
    stockQuantity: newStock,
    pricePerUnit: pricePerUnit
  };

  var options = {
    method: "post",
    contentType: "application/json",
    payload: JSON.stringify(transactionRecord)
  };

  UrlFetchApp.fetch(firebaseUrl + "/transactions.json?auth=" + firebaseSecret, options);

  // Update inventory in Firebase
  var inventoryOptions = {
    method: "patch",
    contentType: "application/json",
    payload: JSON.stringify(inventoryUpdate)
  };

  UrlFetchApp.fetch(firebaseUrl + "/inventory/" + encodeURIComponent(productName) + ".json?auth=" + firebaseSecret, inventoryOptions);

  Logger.log("Transaction recorded and inventory updated.");
}

// Generate Inventory Report
function generateInventoryReport() {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Inventory");
  var data = sheet.getDataRange().getValues();
  var doc = DocumentApp.create("Inventory Report " + new Date().toISOString());
  var body = doc.getBody();

  body.appendParagraph("📊 Inventory Report").setHeading(DocumentApp.ParagraphHeading.HEADING1);
  body.appendParagraph("Generated on: " + new Date().toLocaleString());

  var table = [];
  table.push(["Product Name", "Stock Quantity", "Price Per Unit"]);
  
  for (var i = 1; i < data.length; i++) {
    table.push([data[i][0], data[i][1], "$" + data[i][2]]);
  }
  
  body.appendTable(table);
  var pdf = doc.getAs("application/pdf");
  var adminEmail = "admin@example.com";

  MailApp.sendEmail({
    to: adminEmail,
    subject: "📄 Inventory Report",
    body: "Attached is the latest inventory report.",
    attachments: [pdf]
  });

  Logger.log("Inventory report generated and sent.");
}

// Simple Chat System - Send Message to Firebase
function sendChatMessage(user, message) {
  var firebaseUrl = "YOUR_FIREBASE_DATABASE_URL";
  var firebaseSecret = "YOUR_FIREBASE_SECRET_KEY";

  var chatMessage = {
    user: user,
    message: message,
    timestamp: new Date().toISOString()
  };

  var options = {
    method: "post",
    contentType: "application/json",
    payload: JSON.stringify(chatMessage)
  };

  UrlFetchApp.fetch(firebaseUrl + "/chat.json?auth=" + firebaseSecret, options);

  Logger.log("Chat message sent.");
}
