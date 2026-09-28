(() => {
  if (window.ihAuto?.stop) {
    window.ihAuto.stop();
  }

  const MAX_APPLIES = 1;
  const INTERVAL = 1000;
  const LOAD_TIMEOUT = 15000;
  const APPLY_TIMEOUT = 12000;

  const bot = window.ihAuto = {
    attempts: 0,
    confirmed: 0,
    processed: new Set(),
    pending: null,
    opening: null,
    timer: null,
    stopped: false,

    stop() {
      clearInterval(this.timer);
      this.timer = null;
      this.stopped = true;

      console.log("[Instahyre] Stopped.", {
        attempts: this.attempts,
        confirmed: this.confirmed
      });
    }
  };

  const log = (...args) =>
    console.log("[Instahyre]", ...args);

  const visible = el => {
    if (!el || !el.isConnected) return false;

    const rect = el.getBoundingClientRect();
    const style = getComputedStyle(el);

    return (
      rect.width > 0 &&
      rect.height > 0 &&
      style.display !== "none" &&
      style.visibility !== "hidden" &&
      !el.closest(".ng-hide")
    );
  };

  const first = selector =>
    [...document.querySelectorAll(selector)].find(visible);

  const getModal = () =>
    [...document.querySelectorAll(
      ".candidate-apply-modal"
    )].find(modal =>
      !modal.closest(".ng-hide") &&
      visible(
        modal.querySelector(
          ".app-modal, .application-modal-wrap"
        )
      )
    );

  const getJobId = el => {
    try {
      let scope = angular.element(el).scope();

      while (scope) {
        const opp = scope.opp;

        if (opp) {
          return String(
            opp.id || opp.job?.id || ""
          );
        }

        scope = scope.$parent;
      }
    } catch {}

    return "";
  };

  const getJobName = modal =>
    modal.querySelector(
      ".employer-job-name, .job-title, h2"
    )?.innerText.trim() || "Unknown job";

  const close = () => {
    const modal = getModal();

    const button = modal &&
      [...modal.querySelectorAll(
        ".application-modal-close"
      )].find(visible);

    button?.click();
  };

  function findApply(modal) {
    if (!modal) return null;

    // First try Instahyre's actual Apply control.
    const byAngular = [
      ...modal.querySelectorAll(
        '[ng-click*="submitChoice"]'
      )
    ].find(el =>
      visible(el) &&
      /\bApply\b/i.test(el.innerText)
    );

    if (byAngular) return byAngular;

    // Fallback for buttons with different markup.
    return [
      ...modal.querySelectorAll(
        "button, a, [role='button']"
      )
    ].find(el =>
      visible(el) &&
      /^Apply(?:\s+Now)?$/i.test(
        el.innerText.trim()
      )
    );
  }

  function tick() {
    if (bot.stopped) return;

    const error = first(
      ".growl-item.growl-error"
    );

    if (error) {
      log("Site error:", error.innerText);
      bot.stop();
      return;
    }

    const bulk = first(
      ".candidate-apply-all-modal .app-modal, " +
      ".candidate-apply-all-modal .application-modal-wrap"
    );

    if (bulk) {
      log("Similar-jobs popup detected. Review manually.");
      bot.stop();
      return;
    }

    if (first("#apply-external-modal")) {
      log("External application required.");
      bot.stop();
      return;
    }

    const modal = getModal();

    // Wait for confirmation after clicking Apply.
    if (bot.pending) {
      const pending = bot.pending;

      const sent = modal &&
        [...modal.querySelectorAll(
          "button, a"
        )].some(el =>
          visible(el) &&
          /Application Sent!/i.test(el.innerText)
        );

      const currentApply = modal &&
        findApply(modal);

      const currentId = currentApply
        ? getJobId(currentApply)
        : "";

      const advanced =
        currentId &&
        pending.id &&
        currentId !== pending.id;

      if (sent || advanced) {
        bot.confirmed++;

        log(
          "Application confirmed or advanced:",
          pending.name
        );

        bot.pending = null;

        if (bot.attempts >= MAX_APPLIES) {
          bot.stop();
          return;
        }

      } else if (
        Date.now() - pending.time > APPLY_TIMEOUT
      ) {
        log(
          "Could not verify application. Check Applied jobs:",
          pending.name
        );

        bot.stop();
        return;

      } else {
        return;
      }
    }

    if (bot.attempts >= MAX_APPLIES) {
      bot.stop();
      return;
    }

    // STEP 1: Open a job using View job.
    if (!modal) {
      if (bot.opening) {
        if (
          Date.now() - bot.opening.time <
          LOAD_TIMEOUT
        ) {
          return;
        }

        log(
          "Job details did not open:",
          bot.opening.name
        );

        bot.opening = null;
      }

      const cards = [
        ...document.querySelectorAll(
          "#employer-profile-opportunity"
        )
      ];

      const next = cards.find(card => {
        if (!visible(card)) return false;

        const id = getJobId(card);

        const key =
          id || card.innerText.slice(0, 150);

        if (bot.processed.has(key)) {
          return false;
        }

        return [
          ...card.querySelectorAll(
            "button, a, [role='button']"
          )
        ].some(el =>
          visible(el) &&
          /View job/i.test(el.innerText)
        );
      });

      if (!next) {
        log("No more visible jobs.");
        bot.stop();
        return;
      }

      const id = getJobId(next);

      const key =
        id || next.innerText.slice(0, 150);

      const name =
        next.querySelector(".employer-job-name")
          ?.innerText.trim() ||
        next.innerText.slice(0, 100);

      const view = [
        ...next.querySelectorAll(
          "button, a, [role='button']"
        )
      ].find(el =>
        visible(el) &&
        /View job/i.test(el.innerText)
      );

      bot.processed.add(key);

      bot.opening = {
        id,
        name,
        time: Date.now()
      };

      log("Clicking View job:", name);

      view.click();

      return;
    }

    // STEP 2: Wait for the Apply button.
    const apply = findApply(modal);

    if (!apply) {
      const alreadySent = [
        ...modal.querySelectorAll(
          "button, a"
        )
      ].some(el =>
        visible(el) &&
        /Application Sent!/i.test(el.innerText)
      );

      if (alreadySent) {
        log("Already applied:", getJobName(modal));
        bot.opening = null;
        close();
        return;
      }

      if (!bot.opening) {
        bot.opening = {
          id: "",
          name: getJobName(modal),
          time: Date.now()
        };
      }

      const elapsed =
        Date.now() - bot.opening.time;

      if (elapsed < LOAD_TIMEOUT) {
        return;
      }

      log(
        "Apply button not found after 15 seconds:",
        bot.opening.name
      );

      log(
        "Visible modal buttons:",
        [...modal.querySelectorAll(
          "button, a, [role='button']"
        )]
          .filter(visible)
          .map(el => el.innerText.trim())
      );

      bot.stop();
      return;
    }

    // STEP 3: Apply only after the button appears.
    const id =
      getJobId(apply) ||
      bot.opening?.id;

    const name = getJobName(modal);

    if (!id) {
      log("Could not identify job. Stopping.");
      bot.stop();
      return;
    }

    if (bot.processed.has("applied:" + id)) {
      bot.opening = null;
      close();
      return;
    }

    const required = [
      ...modal.querySelectorAll(
        "input[required], " +
        "textarea[required], " +
        "select[required]"
      )
    ].filter(visible);

    if (required.length) {
      log(
        "Additional information required:",
        name
      );

      bot.stop();
      return;
    }

    bot.processed.add("applied:" + id);

    bot.attempts++;

    bot.pending = {
      id,
      name,
      time: Date.now()
    };

    bot.opening = null;

    log("Clicking Apply:", name);

    apply.click();
  }

  log(
    "Started. Maximum applications:",
    MAX_APPLIES
  );

  bot.timer = setInterval(() => {
    try {
      tick();
    } catch (error) {
      console.error(error);
      bot.stop();
    }
  }, INTERVAL);
})();
