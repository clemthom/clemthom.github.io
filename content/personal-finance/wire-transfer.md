---
title: "Wire transfer using a nostro account"
summary: "Sending wire transfer from etrade using a nostro account" 
date: 2023-12-22
tags: ["wire","rsu","etrade","nostro"]
author: "Clement Thomas"
---

Whenever i sell my RSU stocks in etrade portal,i usually wire them directly to my indian bank account. The following details will usually be enough for a successful wire transfer.
* My Bank's swift code
* My Bank's account number
* My Bank's IFSC code

This time, I was asked to use a nostro account to get a preferential conversion rate by my bank's relationship manager and so i modified the wire instructions in the etrade portal. This time i had the following details to add.

* My Bank's swift code
* My Bank's account number
* My Bank's IFSC code
* Nostro account number
* Routing number of nostro account


But for some reasons the wire transfer didn't go through and i received a wire deposit in my etrade account itself.

![wire_deposit](/img/pf/wire-transfer/wire_deposit.jpg)

The reason being, i filled the account details in reverse order. I filled the nostro account details first and my account details in additional account details section, which was incorrect.

Folks from ICICI remittence team helped to correct the wire instructions, We should fill our account details first and the nostro account details in the additional account details section. The screenshot below is a correct one which worked.

![wire_transfer_instructions](/img/pf/wire-transfer/correct_instr.jpg)

Hope this helps someone trying to wire transfer from etrade using a nostro account.