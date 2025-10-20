---
title: "ارسال ایمیل در صورت تغییر در Google Sheet"
categories:
  - Trick
tags:
  - google_sheet
  - notification
  - email
  - script
---

در صورتی که در Google Sheet نیاز داشتید در صورت Add/Update/Delete متوجه تغییرات بشوید کافی است از اسکریپت زیر استفاده کنید.  
بدین منظور از سربرگ Extensions گزینه Apps Script را انتخاب کنید. سپس توسط + یک فایل جدید ایجاد کنید.  
در فایل ایجاد شده کد زیر را قرار دهید.   

```csharp
function onOpen(e) {
   installTrigger();
};

function listTriggers() {
  const triggers = ScriptApp.getProjectTriggers();
  triggers.forEach(trigger => {
    Logger.log(`Trigger ID: ${trigger.getUniqueId()}, Handler: ${trigger.getHandlerFunction()}`);
  });
}

function deleteTriggers() {
  const triggers = ScriptApp.getProjectTriggers();
  triggers.forEach(trigger => {
    ScriptApp.deleteTrigger(trigger);
  });
}

// Function to install the onEdit trigger
function installTrigger() {
  const triggers = ScriptApp.getProjectTriggers();
  
  // Check if the trigger already exists
  const existingTrigger = triggers.some(trigger => 
    trigger.getHandlerFunction() === 'sendEmailOnEdit'
  );
  
  if (!existingTrigger) {
    ScriptApp.newTrigger('sendEmailOnEdit')
             .forSpreadsheet(SpreadsheetApp.getActiveSpreadsheet())
             .onEdit()
             .create();
  }
}

// Global variable to store processed changes
let processedChanges = [];

// Function to send email on edit
function sendEmailOnEdit(e) {
  // Get the active spreadsheet and sheet
  const sheet = e.source.getActiveSheet();
  const range = e.range;
  const row = range.getRow();
  const column = range.getColumn();
  const oldValue = e.oldValue;
  const newValue = e.value;
  const changeType = getChangeType(e);
  const headerRange = sheet.getRange(1,1,1,sheet.getLastColumn()).getValues()[0];
  const header = headerRange[column-1];

  if (header !== 'توضیح' || newValue === oldValue) {
    return;
  }

  // Check if the change has already been processed
  const changeId = `${sheet.getName()}_${row}_${column}_${newValue}`;
  const cache = CacheService.getScriptCache();
  const cachedChange = cache.get(changeId);
  
  if (cachedChange) {
    return;
  }

  // Get all user emails with access to the spreadsheet
  const userEmails = getAllUserEmails();

  // Create the email subject and body
  const subject = 'Life';

    const body = `
    <div style="direction: rtl; text-align: right;font-family: Tahoma;">
      <p>نوع تغییر: ${changeType}</p>
      <p>نام شیت: ${sheet.getName()}</p>
      <p>ردیف: ${row}</p>
      <p>مقدار: ${newValue}</p>
    </div>
  `;             

  //Send the email to all users
  userEmails.forEach(email => {
    MailApp.sendEmail({
      to: email,
      subject: subject,
      htmlBody: body
    });
 });

  // Cache the changeId to avoid sending duplicate emails
  cache.put(changeId, 'processed', 3600); // Cache it for 1 hour
}

// Function to determine the type of change
function getChangeType(e) {
  const range = e.range;
  const sheet = e.source.getActiveSheet();
  const lastRow = sheet.getLastRow();
  const lastColumn = sheet.getLastColumn();

  if (range.getRow() > lastRow - 1 && range.getLastRow() == lastRow) {
    return 'جدید';
  } else if (e.oldValue === undefined) {
    return 'حذف';
  } else {
    return 'ویرایش';
  }
}

// Function to get all user emails with access to the spreadsheet
function getAllUserEmails() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet();
  const editors = sheet.getEditors();
  
  const emails = new Set();

  editors.forEach(editor => emails.add(editor.getEmail()));

  return Array.from(emails);
}
```

این کد ایمیل تمام افرادی که به فایل دسترسی دارند را بدست می‌آوردن و در صورت تغییر به آنها ایمیل ارسال می‌کند.  
اکنون بر روی Save کلیک کنید. سپس از همان بالا که لیست متودها را نشان می‌دهد یکبار OpOpen را انتخاب کنید و سپس ر روی Run کلیک کنید.  
با اینکار پنجره‌ای برای گرفتن دسترسی باز می‌شود که باید آن را تایید کنید. همچنین با توجه به اینکه حساب شما تایید شده توسط گوگل نیست نیز ممکن است هشدار داده شود که آن را نیز می‌توانید با unsafe تایید کنید.  

در کد بالا از cache استفاده شده است تا از ارسال چندباره جلوگیری شود. همچنین زمان install trigger نیز این مورد بررسی شده است.  
توسط listTriggers و deleteTriggers می‌توانید کد خود را دیباگ کنید تا اگر از قبل تریگری وجود داشت حذف شود تا ایمیل چندبار ارسال نشود.  
همچنین فقط تغییرات ستون با عنوان توضیح در این کد مدیریت شده است تا ایمیل برای آن ارسال شود که می‌توانید آن را تغییر دهید.  