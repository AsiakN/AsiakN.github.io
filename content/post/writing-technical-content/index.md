---
title: Why Verify Webhooks
date: 2024-05-03
math: false
image:
  placement: 2
  # caption: 'Image credit: [**John Moeses Bauan**](https://unsplash.com/photos/OGZtQF8iC0g)'
---


Webhooks are a potential security loophole for many organizations. **Why?** Because of how they work. A webhook is pretty much a POST request from an unknown source.

The problems with webhook would be reduced very much if we knew the source of the POST requests and we trusted that those organizations would not be malicious. But hey, don't outsource your security and hope for the best. It's the hope that kills!

## What to do about this?
It's actually counterintuitive... you should verify the webhooks. Yes, check that the request is from who it says it's from. Simple right? yes, that's it.

### How do I check for this?

I'll leave the specific implementation details of this check to you. But on a top level, you should look out for two things. There are more security implications you should look at. 
 **verify the signature of webhook. Most providers sign their webhooks with a secret key that could be used to verify that origin of the request. 

* verify the signature of webhook. Most providers sign their webhooks with a secret key that could be used to verify that origin of the request. 

*  But there's another angle. Attackers may intercept the original payload(with valid signature) before transmitting it you. Such a request will pass the signature verification step. This type of attack is called a [Replay Attack](https://en.wikipedia.org/wiki/Replay_attack). However there's a remedy. 
   * Timestamp verification. Verify that the time between when the payload was sent and the time it was received is not abnormally long. 

That's a bit about webhook verification. These notes are just for me to be sure I can read them some time in the future. 

### Code
coming soon

    ```Go
    import (
    svix "github.com/svix/svix-webhooks/go"
  )
    ```

<!-- renders as

```python
import pandas as pd
data = pd.read_csv("data.csv")
data.head()
``` -->

<!-- ### Did you find this page helpful? Consider sharing it 🙌 -->
