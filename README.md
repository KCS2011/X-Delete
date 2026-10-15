Script to delete all X / Twitter tweets. (Credits go to https://github.com/techleadhd, who let ChatGPT create the first version.). 

Updated by Claude to add a MAX_AGE_DAYS to keep some recents posts and some rate limiting to prevent Too Many Requests (429)

### How to run:
- Open Chrome and go to your X profile (https://x.com/USERNAME/with_replies). Make sure to scroll down as far as possible before running the script.
- Open Chrome Developer Console. Copy/Paste the code
- If needed adjust the MAX_AGE_DAYS setting
- Run!

```
const deleteAllTweets = async () => {
  // ===== Settings =====
  const MAX_AGE_DAYS = 7;              // only remove tweets older than this
  const DELETE_DELAY = 4000;           // pause after each deletion (ms)
  const BATCH_SIZE = 50;               // after this many deletions...
  const BATCH_PAUSE = 5 * 60000;       // ...take a 5-minute break
  const RATE_LIMIT_PAUSE = 15 * 60000; // first pause after a 429 (doubles each time, max 2h)
  const EMPTY_RETRIES = 3;             // scroll attempts with no new tweets before stopping
  const DEBUG = true;                  // false = less console output
  // ====================

  window.stopDelete = false; // type `stopDelete = true` in the console to stop

  const selectors = {
    tweet: '[data-testid="tweet"]',
    caret: '[data-testid="caret"]',
    userName: '[data-testid="User-Name"] a[href^="/"]',
    profileLink: '[data-testid="AppTabBar_Profile_Link"]',
    accountSwitcher: '[data-testid="SideNav_AccountSwitcher_Button"]',
    menuItem: '[role="menuitem"]',
    deleteConfirm: '[data-testid="confirmationSheetConfirm"]',
    unretweet: '[data-testid="unretweet"]',
    unretweetConfirm: '[data-testid="unretweetConfirm"]'
  };

  const cutoff = Date.now() - MAX_AGE_DAYS * 86400000;
  const processed = new Set();
  let deleted = 0, skipped = 0, rateLimited = false, backoff = RATE_LIMIT_PAUSE;

  const log = (...a) => DEBUG && console.log(...a);

  const delay = ms => {
    const jitter = ms * 0.25;
    return new Promise(r => setTimeout(r, ms + (Math.random() * jitter * 2 - jitter)));
  };

  // Wait with a countdown in the console, once per minute (stoppable)
  const countdown = async (ms, label) => {
    const end = Date.now() + ms;
    while (Date.now() < end) {
      if (window.stopDelete) return;
      const left = Math.ceil((end - Date.now()) / 60000);
      console.log(`${label}: ${left} minute${left === 1 ? '' : 's'} left...`);
      await new Promise(r => setTimeout(r, Math.min(60000, end - Date.now())));
    }
    console.log(`${label} over, continuing.`);
  };

  // ----- Pre-checks -----
  // Find your handle: 1) profile link, 2) account switcher, 3) the page URL
  const getMyHandle = () => {
    const link = document.querySelector(selectors.profileLink)?.getAttribute('href');
    if (link) return link;

    const switcher = document.querySelector(selectors.accountSwitcher);
    const m = switcher?.textContent.match(/@(\w+)/);
    if (m) return '/' + m[1];

    const fromUrl = location.pathname.split('/')[1];
    if (fromUrl && !['home', 'explore', 'notifications', 'messages', 'i', 'settings', 'search'].includes(fromUrl)) {
      return '/' + fromUrl;
    }
    return null;
  };

  const handle = getMyHandle();
  if (!handle) {
    console.error('Could not determine your username. Open your own profile (x.com/yourname/with_replies) and try again.');
    return;
  }
  const myHandle = handle.toLowerCase();
  console.log(`Detected account: ${myHandle}`);

  // Wait up to 30 seconds for tweets to appear
  for (let i = 0; i < 15 && !document.querySelector(selectors.tweet); i++) {
    if (i === 0) console.log('Waiting for tweets to load...');
    await delay(2000);
  }
  if (!document.querySelector(selectors.tweet)) {
    console.error('No tweets found on this page. Is the timeline loaded (no "Something went wrong")?');
    return;
  }

  // ----- 429 detection (only on delete/unretweet requests) -----
  const isDeleteCall = url => /DeleteTweet|DeleteRetweet|unretweet/i.test(url || '');
  const origFetch = window.fetch;
  window.fetch = async (...args) => {
    const res = await origFetch(...args);
    const url = typeof args[0] === 'string' ? args[0] : args[0]?.url;
    if (res.status === 429 && isDeleteCall(url)) rateLimited = true;
    return res;
  };
  const origOpen = XMLHttpRequest.prototype.open;
  XMLHttpRequest.prototype.open = function (method, url, ...rest) {
    this.addEventListener('load', () => {
      if (this.status === 429 && isDeleteCall(url)) rateLimited = true;
    });
    return origOpen.call(this, method, url, ...rest);
  };

  const waitIfRateLimited = async () => {
    if (!rateLimited) return;
    console.warn(`429 on delete. Pausing ${Math.round(backoff / 60000)} min (from ${new Date().toLocaleTimeString()})...`);
    await countdown(backoff, 'Rate limit pause');
    backoff = Math.min(backoff * 2, 2 * 3600000);
    rateLimited = false;
  };

  // ----- Helpers -----
  const getTweetDate = tweet => {
    const time = tweet?.querySelector('time[datetime]'); // first = outer tweet, not a quoted one
    return time ? new Date(time.getAttribute('datetime')) : null;
  };

  const isOwnTweet = tweet =>
    tweet?.querySelector(selectors.userName)?.getAttribute('href').toLowerCase() === myHandle;

  const closeMenu = () =>
    document.body.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', bubbles: true }));

  const getButtons = () =>
    Array.from(document.querySelectorAll(`${selectors.tweet} ${selectors.caret}`))
      .filter(b => !processed.has(b));

  const scrollToEnd = () =>
    window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' });

  // ----- Actions -----
  const doUnretweet = async (tweet, date) => {
    const btn = tweet.querySelector(selectors.unretweet);
    if (!btn) return false;
    btn.click();
    await delay(500);
    const confirm = document.querySelector(selectors.unretweetConfirm);
    if (!confirm) { log('Unretweet clicked, but no confirm button found'); closeMenu(); return false; }
    confirm.click();
    await delay(1500);
    if (rateLimited) return null; // retry later
    deleted++;
    console.log(`#${deleted} Unretweeted post from ${date.toLocaleDateString()}`);
    return true;
  };

  const doDelete = async (button, date) => {
    button.click();
    await delay(500);
    const menuItems = Array.from(document.querySelectorAll(selectors.menuItem));
    const deleteOption = menuItems.find(item =>
      item.textContent.includes('Delete') || item.textContent.includes('Verwijder')
    );
    if (!deleteOption) {
      log('No Delete option. Menu items:', menuItems.map(i => i.textContent.trim()));
      closeMenu();
      await delay(300);
      return false;
    }
    deleteOption.click();
    await delay(500);
    const confirm = document.querySelector(selectors.deleteConfirm);
    if (!confirm) { log('Delete clicked, but no confirm button found'); closeMenu(); return false; }
    confirm.click();
    await delay(1500);
    if (rateLimited) return null; // retry later
    deleted++;
    console.log(`#${deleted} Deleted tweet from ${date.toLocaleDateString()}`);
    return true;
  };

  const processTweet = async button => {
    processed.add(button);
    const tweet = button.closest(selectors.tweet);
    const date = getTweetDate(tweet);

    if (!date) { skipped++; log('Skipped: no date (ad/promoted?)'); return false; }
    if (date.getTime() > cutoff) {
      skipped++; log(`Skipped: ${date.toLocaleDateString()} is newer than ${MAX_AGE_DAYS} days`);
      return false;
    }

    const own = isOwnTweet(tweet);
    const isRetweet = !!tweet.querySelector(selectors.unretweet);

    if (!own && !isRetweet) {
      skipped++; log('Skipped: not your tweet (conversation context)');
      return false;
    }

    try {
      // Own tweet → delete; someone else's tweet you retweeted → unretweet
      const result = own ? await doDelete(button, date) : await doUnretweet(tweet, date);
      if (result === null) { processed.delete(button); return false; } // 429: try again later
      return result;
    } catch (err) {
      console.error('Error:', err);
      return false;
    }
  };

  // ----- Main loop -----
  console.log(`Removing tweets by ${myHandle} older than ${new Date(cutoff).toLocaleString()}`);
  let emptyRounds = 0;

  try {
    while (!window.stopDelete) {
      await waitIfRateLimited();

      const buttons = getButtons();
      if (!buttons.length) {
        if (++emptyRounds > EMPTY_RETRIES) break;
        log(`No new tweets, scrolling (${emptyRounds}/${EMPTY_RETRIES})...`);
        scrollToEnd();
        await delay(5000);
        continue;
      }
      emptyRounds = 0;

      for (const button of buttons) {
        if (window.stopDelete) break;
        await waitIfRateLimited();
        if (await processTweet(button)) {
          await delay(DELETE_DELAY);
          if (deleted % BATCH_SIZE === 0) {
            console.log(`Batch of ${BATCH_SIZE} done, pausing ${BATCH_PAUSE / 60000} min...`);
            await countdown(BATCH_PAUSE, 'Batch pause');
          }
        }
      }

      scrollToEnd();
      await delay(3000);
    }
    if (window.stopDelete) console.log('Stopped by user.');
  } finally {
    window.fetch = origFetch;
    XMLHttpRequest.prototype.open = origOpen;
    console.log(`Done. Removed: ${deleted}, skipped: ${skipped}`);
  }
};

deleteAllTweets().catch(err => console.error('Script failed:', err));
```
<img width="813" alt="Screenshot 2023-12-31 at 7 18 45 PM" src="https://github.com/techleadhd/XDelete/assets/61847557/473165c5-9b7c-4065-98fd-5856fcbfb3a8">
