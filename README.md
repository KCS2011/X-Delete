Script to delete all X / Twitter tweets. (Credits go to https://github.com/techleadhd, who let ChatGPT create the first version.). 

Updated by Claude to add a MAX_AGE_DAYS to keep some recents posts and some rate limiting to prevent Too Many Requests (429)

### How to run:
- Open Chrome and go to your X profile (https://x.com/USERNAME/with_replies)
- Open Chrome Developer Console. Copy/Paste the code
- If needed adjust the MAX_AGE_DAYS setting
- Run!

```
const deleteAllTweets = async () => {
  const MAX_AGE_DAYS = 7;
  const DELETE_DELAY = 4000;        // pause after each deletion (ms)
  const BATCH_SIZE = 50;            // after this many deletions...
  const BATCH_PAUSE = 5 * 60000;    // ...take a 5-minute break
  const RATE_LIMIT_PAUSE = 15 * 60000; // first pause after a 429 (doubles each time)

  const cutoff = Date.now() - MAX_AGE_DAYS * 86400000;
  const processed = new Set();
  const selectors = {
    tweet: '[data-testid="tweet"]',
    caret: '[data-testid="caret"]',
    menuItem: '[role="menuitem"]',
    deleteConfirm: '[data-testid="confirmationSheetConfirm"]',
    unretweet: '[data-testid="unretweet"]',
    unretweetConfirm: '[data-testid="unretweetConfirm"]'
  };

  let deleted = 0, skipped = 0, rateLimited = false, backoff = RATE_LIMIT_PAUSE;

  // --- Detect 429 responses from X's own requests ---
  const origFetch = window.fetch;
  window.fetch = async (...args) => {
    const res = await origFetch(...args);
    if (res.status === 429) rateLimited = true;
    return res;
  };
  const origOpen = XMLHttpRequest.prototype.open;
  XMLHttpRequest.prototype.open = function (...args) {
    this.addEventListener('load', () => { if (this.status === 429) rateLimited = true; });
    return origOpen.apply(this, args);
  };

  const delay = ms => {
    const jitter = ms * 0.25;
    return new Promise(r => setTimeout(r, ms + (Math.random() * jitter * 2 - jitter)));
  };

  const waitIfRateLimited = async () => {
    if (!rateLimited) return;
    const mins = Math.round(backoff / 60000);
    console.warn(`429 rate limit hit. Pausing ${mins} min (at ${new Date().toLocaleTimeString()})...`);
    await delay(backoff);
    backoff = Math.min(backoff * 2, 2 * 3600000); // max 2 hours
    rateLimited = false;
    console.log('Resuming...');
  };

  const getTweetDate = tweet => {
    const time = tweet?.querySelector('time[datetime]');
    return time ? new Date(time.getAttribute('datetime')) : null;
  };

  const closeMenu = () =>
    document.body.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', bubbles: true }));

  const getButtons = () =>
    Array.from(document.querySelectorAll(`${selectors.tweet} ${selectors.caret}`))
      .filter(b => !processed.has(b));

  const scrollToEnd = () =>
    window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' });

  const attemptDelete = async button => {
    processed.add(button);
    const tweet = button.closest(selectors.tweet);
    const date = getTweetDate(tweet);
    if (!date || date.getTime() > cutoff) { skipped++; return false; }

    try {
      button.click();
      await delay(400);

      const deleteOption = Array.from(document.querySelectorAll(selectors.menuItem))
        .find(item => /^(Delete|Verwijderen)$/i.test(item.textContent.trim()));

      if (deleteOption) {
        deleteOption.click();
        await delay(400);
        const confirm = document.querySelector(selectors.deleteConfirm);
        if (confirm) {
          confirm.click();
          await delay(1500);
          if (rateLimited) { processed.delete(button); return false; } // retry later
          deleted++;
          console.log(`#${deleted} Deleted tweet from ${date.toLocaleDateString()}`);
          return true;
        }
      } else {
        closeMenu();
        await delay(300);
      }

      const unretweet = tweet?.querySelector(selectors.unretweet);
      if (unretweet) {
        unretweet.click();
        await delay(400);
        const confirm = document.querySelector(selectors.unretweetConfirm);
        if (confirm) {
          confirm.click();
          await delay(1500);
          if (rateLimited) { processed.delete(button); return false; }
          deleted++;
          console.log(`#${deleted} Unretweeted post from ${date.toLocaleDateString()}`);
          return true;
        }
      }
    } catch (err) {
      console.error('Error:', err);
    }
    return false;
  };

  console.log(`Removing tweets older than ${new Date(cutoff).toLocaleString()}`);

  while (true) {
    await waitIfRateLimited();

    const buttons = getButtons();
    if (!buttons.length) {
      scrollToEnd();
      await delay(5000);
      if (!getButtons().length) break;
      continue;
    }

    for (const button of buttons) {
      await waitIfRateLimited();
      const acted = await attemptDelete(button);
      if (acted) {
        await delay(DELETE_DELAY);
        if (deleted % BATCH_SIZE === 0) {
          console.log(`Batch of ${BATCH_SIZE} done, pausing ${BATCH_PAUSE / 60000} min...`);
          await delay(BATCH_PAUSE);
        }
      }
    }

    scrollToEnd();
    await delay(3000);
  }

  window.fetch = origFetch;
  XMLHttpRequest.prototype.open = origOpen;
  console.log(`Done. Removed: ${deleted}, skipped: ${skipped}`);
};

deleteAllTweets().catch(err => console.error('Script failed:', err));
```
<img width="813" alt="Screenshot 2023-12-31 at 7 18 45 PM" src="https://github.com/techleadhd/XDelete/assets/61847557/473165c5-9b7c-4065-98fd-5856fcbfb3a8">
