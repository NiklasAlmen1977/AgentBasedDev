# Aaro upload file - user guide

For the GFOs who upload monthly actuals to Aaro.

The Excel file `Example SEND RETRIEVE DIFF.xlsx` fetches the Aaro export for one company and
one accounting period from Power BI (semantic model `Aaro endpoint (sm)`). You only see the
companies you have access to: Power BI checks this for you when the file fetches the data.

> Draft prepared with AI assistance (Claude). Validation by Kristoffer Rapp (owner of the Aaro
> export solution) and Hanna Sofia (Head of Analytics) is pending. The file name can change after
> that validation.

## What you need

- Access to Aaro endpoint (sm), which is two things:
  - **Build permission** on the semantic model, so that Excel can connect to it.
  - Your companies in the **Aaro access list** (row-level security), so that you see their data.

  If you don't have access yet, or the company list in the file is empty, send a ticket to
  servicedesk@vitrolifegroup.com for Group Analytics, with your manager for approval.
- Excel desktop (Microsoft 365), signed in with your Vitrolife account.
- The Aaro Excel add-in, as today.

## Step by step

1. Open the file and choose **Open in Desktop App**. Save your own copy if you want to keep it;
   don't overwrite the original file.
2. If Excel shows a yellow bar, choose **Enable Content**. The first time, sign in with your
   Vitrolife account (Organizational account) if Excel asks.
3. On the sheet **Aaro_Upload**, choose your **Company** (cell C2) and your **Accounting period**
   (cell C3) from the dropdowns. Choose exactly one of each.
4. Press **Ctrl+Alt+F5** (or **Data → Refresh All**) to fetch the values.
5. Look at **Check** (cell B4):

   | Check shows | What it means | What to do |
   |---|---|---|
   | ✓ (green) "... rows loaded for ..." | The table has the rows for your company and period | Continue |
   | ⚠ (red) "The selection has changed" | You changed the company or period after the last refresh | Press Ctrl+Alt+F5 again |
   | "No data for ..." (red) | There are no amounts for that company and period | Check the period, or ask Group Analytics |
   | "Choose a company and an accounting period" | One of the two is empty | Choose both (step 3) |

6. Check the amounts in `actuals`. They are already summed (balance sheet from the start,
   P&L from January to the period) and have the sign Aaro expects. Don't change the signs.
7. Use the Aaro add-in as before. **SEND formula** is what will be sent to Aaro,
   **RETRIEVE formula** is what is in Aaro now, and **Diff** is the difference between them.
8. Upload with the Aaro add-in.

To change company or period: choose the new values in C2 and C3, refresh (Ctrl+Alt+F5),
and check that B4 shows ✓ again before you upload.

## Excel in the browser

Fetching the values from Power BI is only tested in Excel desktop. If you open the file in the
browser and refreshing doesn't work, choose **Editing → Open in Desktop App** and continue there.

## Support

servicedesk@vitrolifegroup.com, for Group Analytics.
