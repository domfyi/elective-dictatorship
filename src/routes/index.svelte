<script>
  import moment from "moment";
  import rawActs from "../data/acts.js";
  import rawGovs from "../data/govs.js";

  let y = 0;
  let height = 800;
  let innerWidth = 800;

  let govs = false;
  let acts = false;

  let currentPM = 0;
  let currentDate = moment(new Date()).format("MMM YYYY");
  let currentSeats = 0;
  let currentMajority = 0;
  let currentAct = "";
  let currentActName = "";
  let currentActLink = "";
  let currentActDate = "";

  let majorities = [];

  const getActs = () => {
    return rawActs.reduce((list, act) => {
      if (!act.Date || !act.Date.length) return list;
      const key = act.Date.slice(0, 7);
      if (act["Visible"] !== "TRUE") return list;
      if (!list[key]) list[key] = [];
      list[key].push(act);
      return list;
    }, {});
  };

  const getGovs = () => {
    const govMap = rawGovs.reduce(
      (list, { name, nickname, party, image, date, majority, seats, coalition }) => {
        if (!list[name]) {
          list[name] = { name, nickname, party, image, majority: [] };
        }
        list[name].majority.push({ date, majority, seats, coalition });
        return list;
      },
      {}
    );
    return Object.values(govMap);
  };

  const parties = {
    CON: "rgb(0, 144, 235)",
    LAB: "rgb(227, 7, 2)",
    LIB: "rgb(232, 160, 0)",
    WHIG: "rgb(198, 140, 0)",
    TORY: "rgb(0, 100, 180)"
  };

  acts = getActs();
  govs = getGovs();
  currentSeats = govs[0].majority[0].seats;
  currentMajority = govs[0].majority[0].majority;
  majorities = [
    ...[...govs]
      .filter(pm => pm.majority && pm.majority.length)
      .map(pm => pm.majority)
  ];

  function monthDiff(dateFrom, dateTo) {
    return (
      dateTo.getMonth() -
      dateFrom.getMonth() +
      12 * (dateTo.getFullYear() - dateFrom.getFullYear())
    );
  }

  const getMajorityEndDate = (i, i_m) => {
    let start;
    if (majorities[i][i_m - 1]) {
      start = majorities[i][i_m - 1];
    } else {
      start = [...majorities[i - 1]].pop();
    }
    return start ? start.date : false;
  };

  const getMajorityDateRange = (i, i_m) => {
    if (!majorities[i][i_m]) return { diff: 0 };
    const start = new Date(majorities[i][i_m].date);
    const end =
      i + i_m === 0 ? new Date() : new Date(getMajorityEndDate(i, i_m));
    if (!start) return { diff: 0 };
    const diff = monthDiff(new Date(start), new Date(end));
    return { i, i_m, end, start, diff: Math.max(0, diff) };
  };

  const calculateMajorityWidth = ({ majority, seats }) => {
    return ((seats / 2 + majority) / seats) * 100;
  };

  // ── Grid system ──
  const PLAIN_WEIGHT = 6;
  const EVENT_WEIGHT = 70;

  let gridItems = [];
  let scrollMap = [];
  let totalScrollWeight = 0;
  let activeIndex = -1;
  let gridOffset = 0;
  let gridSection;
  let gridInner;
  let gridCols = 30;

  const buildGrid = () => {
    const items = [];
    for (let i = 0; i < govs.length; i++) {
      const pm = govs[i];
      items.push({
        isSeparator: true,
        pmIndex: i,
        party: pm.party,
        partyColor: parties[pm.party],
        pmImage: pm.image,
        pmName: pm.nickname || pm.name
      });
      for (let i_m = 0; i_m < pm.majority.length; i_m++) {
        const range = getMajorityDateRange(i, i_m);
        if (!range.diff) continue;
        for (let i_m_m = 0; i_m_m < range.diff; i_m_m++) {
          const monthKey = moment(range.end)
            .subtract(i_m_m, "months")
            .format("YYYY-MM");
          const monthLabel = moment(range.end)
            .subtract(i_m_m, "months")
            .format("MMM YYYY");
          const monthActs = acts[monthKey] || [];
          items.push({
            monthKey,
            monthLabel,
            pmIndex: i,
            party: pm.party,
            partyColor: parties[pm.party],
            isEvent: monthActs.length > 0,
            acts: monthActs,
            majority: pm.majority[i_m],
            pmName: pm.nickname || pm.name
          });
        }
      }
    }
    const map = [];
    let cum = 0;
    for (const item of items) {
      map.push(cum);
      if (!item.isSeparator) {
        cum += item.isEvent ? EVENT_WEIGHT : PLAIN_WEIGHT;
      }
    }
    gridItems = items;
    scrollMap = map;
    totalScrollWeight = cum;
  };

  buildGrid();

  const GRID_SCROLL_DELAY = 0.25;

  let mobileListHeight = gridItems.length * 10;
  let mobilePositions = [];

  const ensureMobilePositions = () => {
    if (mobilePositions.length || !gridInner) return;
    mobileListHeight = gridInner.scrollHeight;
    const children = gridInner.children;
    for (let i = 0; i < children.length; i++) {
      mobilePositions.push(children[i].offsetTop);
    }
  };

  $: sectionHeight = innerWidth <= 768
    ? mobileListHeight * 2 + height
    : totalScrollWeight + height;

  const scrollToItem = (idx) => {
    if (!gridSection || !scrollMap[idx] && idx !== 0) return;
    const sectionTop = gridSection.offsetTop;
    const isMob = typeof window !== 'undefined' && window.innerWidth <= 768;
    const sectionRange = gridSection.offsetHeight - height;
    const targetScroll = isMob && totalScrollWeight > 0
      ? (scrollMap[idx] / totalScrollWeight) * sectionRange
      : scrollMap[idx];
    const targetY = sectionTop + targetScroll + 20;
    window.scrollTo(0, targetY);
  };

  // ── Scroll → active grid item mapping ──
  $: if (gridSection && scrollMap.length && y !== undefined && height) {
    const sectionTop = gridSection.offsetTop;
    const isMobile = typeof window !== 'undefined' && window.innerWidth <= 768;
    const scrollInSection = y - sectionTop - 20;

    if (scrollInSection < 0) {
      activeIndex = -1;
      currentPM = 0;
      currentDate = moment(new Date()).format("MMM YYYY");
      currentMajority = govs[0].majority[0].majority;
      currentSeats = govs[0].majority[0].seats;
      currentAct = false;
      currentActName = false;
      currentActLink = false;
      currentActDate = false;
      gridOffset = 0;
    } else {
      const sectionScrollRange = gridSection.offsetHeight - height;

      if (isMobile && gridInner) {
        ensureMobilePositions();
        const listH = gridInner.scrollHeight;
        const vpH = gridInner.parentElement.clientHeight;
        const maxOffset = Math.max(0, listH - vpH);
        const progress = sectionScrollRange > 0
          ? Math.max(0, Math.min(1, scrollInSection / sectionScrollRange))
          : 0;
        gridOffset = progress * maxOffset;

        const cursorPos = gridOffset + 16;
        let lo = 0, hi = mobilePositions.length - 1;
        while (lo < hi) {
          const mid = (lo + hi + 1) >> 1;
          if (mobilePositions[mid] <= cursorPos) lo = mid;
          else hi = mid - 1;
        }
        if (gridItems[lo] && gridItems[lo].isSeparator) {
          lo = Math.min(lo + 1, gridItems.length - 1);
        }
        activeIndex = lo;
      } else {
        const itemScroll = scrollInSection;
        let lo = 0, hi = scrollMap.length - 1;
        while (lo < hi) {
          const mid = (lo + hi + 1) >> 1;
          if (scrollMap[mid] <= itemScroll) lo = mid;
          else hi = mid - 1;
        }
        activeIndex = lo;

        if (gridInner) {
          const step = 35;
          const gap = 3;
          const padTop = 16;
          const padBottom = 16;
          gridCols = Math.max(1, Math.floor((gridInner.clientWidth + step - 32) / step));
          const viewportH = height - padTop - padBottom;
          const totalRows = Math.ceil(gridItems.length / gridCols);
          const totalGridHeight = totalRows * step - gap;
          const maxOffset = Math.max(0, totalGridHeight - viewportH);
          const rawProgress = totalScrollWeight > 0
            ? Math.max(0, Math.min(1, scrollInSection / totalScrollWeight))
            : 0;
          const progress = Math.max(0, (rawProgress - GRID_SCROLL_DELAY) / (1 - GRID_SCROLL_DELAY));
          gridOffset = progress * maxOffset;
        }
      }

      const item = gridItems[activeIndex];
      if (item && !item.isSeparator) {
        currentPM = item.pmIndex;
        currentDate = item.monthLabel || currentDate;
        currentMajority = item.majority ? item.majority.majority : currentMajority;
        currentSeats = item.majority ? item.majority.seats : currentSeats;
        if (item.isEvent && item.acts && item.acts.length > 0) {
          const act = item.acts[0];
          currentActName = act.Act;
          currentActLink = act.Link;
          currentAct = act.Simple;
          currentActDate = item.monthLabel;
        } else {
          currentAct = false;
          currentActName = false;
          currentActLink = false;
          currentActDate = false;
        }
      }
    }
  }
</script>

<style>
  main {
    width: 100%;
    text-align: center;
    margin: 0 auto;
  }

  .main-inner {
    background: #f0f0f4;
    display: flex;
    min-height: 100vh;
  }


  /* ── Sidebar ── */
  .sidebar {
    position: fixed;
    top: 0;
    left: 0;
    bottom: 0;
    width: 380px;
    z-index: 100;
    transition: background 0.3s;
    display: flex;
    flex-direction: column;
  }

  .sidebar-inner {
    display: grid;
    grid-template-rows: 1fr auto 1fr;
    align-items: center;
    justify-items: center;
    height: 100%;
    padding: 2rem;
    text-align: center;
  }

  .sidebar-inner.active {
    grid-template-rows: 0.3fr auto 1fr;
  }

  .sidebar-hero {
    grid-row: 2;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1rem;
  }

  .sidebar-flag {
    height: 80px;
    width: 80px;
    border-radius: 10px;
    object-fit: cover;
    object-position: center;
  }

  .sidebar-title {
    font-family: "Big Shoulders Display", cursive;
    font-size: 1.8rem;
    font-weight: 700;
    color: rgba(255, 255, 255, 0.8);
    text-transform: uppercase;
    letter-spacing: 3px;
  }

  .sidebar-avatar {
    width: 180px;
    height: 180px;
    border-radius: 50%;
    object-fit: cover;
    object-position: top;
    border: 3px solid rgba(255, 255, 255, 0.25);
    flex-shrink: 0;
  }

  .sidebar-name {
    font-family: "Big Shoulders Display", cursive;
    font-size: 2.2rem;
    font-weight: 700;
    color: rgba(255, 255, 255, 0.5);
    text-transform: uppercase;
    letter-spacing: 1px;
    line-height: 1.1;
  }

  .sidebar-date {
    font-family: "Big Shoulders Display", cursive;
    font-weight: 900;
    text-transform: uppercase;
    line-height: 1;
    color: rgba(255, 255, 255, 0.5);
    font-size: 1.4rem;
    letter-spacing: 1px;
  }

  .sidebar-quote {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    text-align: center;
    padding: 0.5rem 0;
  }

  .sidebar-quote .quote-text {
    font-family: "Big Shoulders Display", cursive;
    font-size: 3rem;
    font-weight: 700;
    color: rgba(255, 255, 255, 0.4);
    font-style: italic;
    line-height: 1.25;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }

  .sidebar-quote strong {
    font-weight: 900;
    color: rgba(255, 255, 255, 0.6);
  }

  .sidebar-quote .quote-attr {
    font-family: "Big Shoulders Display", cursive;
    font-size: 1.6rem;
    font-weight: 900;
    font-style: normal;
    color: rgba(255, 255, 255, 0.25);
    text-transform: uppercase;
    letter-spacing: 1px;
  }

  .sidebar-act-slot {
    grid-row: 3;
    align-self: start;
    width: 100%;
    padding-top: 1.5rem;
  }

  .sidebar-act {
    display: flex;
    flex-direction: column;
    align-items: center;
    text-decoration: none;
    gap: 0.5rem;
  }

  .act-simple {
    font-family: "Big Shoulders Display", cursive;
    font-size: 2.4rem;
    font-weight: 700;
    color: #fff;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    line-height: 1.1;
  }

  .act-formal {
    font-size: 0.8rem;
    color: rgba(255, 255, 255, 0.45);
    line-height: 1.4;
  }

  /* ── Content area ── */
  .content {
    margin-left: 380px;
    flex: 1;
    min-width: 0;
  }

  /* ── Grid section ── */
  .grid-section {
    position: relative;
    background: #f0f0f4;
  }

  .grid-viewport {
    position: sticky;
    top: 0;
    height: 100vh;
    overflow: hidden;
    display: flex;
    justify-content: center;
    align-items: flex-start;
    padding: 1rem 3px 3px;
  }

  .grid-inner {
    display: grid;
    grid-template-columns: repeat(auto-fill, 32px);
    gap: 3px;
    width: 100%;
    justify-content: center;
    will-change: transform;
  }

  .g {
    width: 32px;
    height: 32px;
    border-radius: 3px;
    background: var(--c);
    opacity: 0.12;
    transition: opacity 0.1s, transform 0.1s;
  }

  .g.sep {
    opacity: 1;
    background: transparent;
    overflow: hidden;
    border-radius: 3px;
  }

  .g.sep img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    object-position: top;
    display: block;
  }

  .g.ev {
    cursor: pointer;
  }

  .g.ev:hover {
    opacity: 0.8;
    transform: scale(1.3);
    z-index: 1;
  }

  .g.ev {
    opacity: 0.45;
  }

  .g.lit {
    opacity: 0.22;
  }

  .g.ev.lit {
    opacity: 1;
  }

  .g.active {
    z-index: 2;
    opacity: 0.35;
    box-shadow: 0 0 0 2px #333, 0 0 8px rgba(0, 0, 0, 0.12);
  }

  .g.ev.active {
    opacity: 1;
    box-shadow: 0 0 0 2px #333, 0 0 8px rgba(0, 0, 0, 0.2);
  }

  .g .g-label {
    display: none;
  }


  /* ── Mobile: top half sidebar, bottom half list ── */
  @media (max-width: 768px) {
    .sidebar {
      position: fixed;
      top: 0;
      left: 0;
      right: 0;
      bottom: auto;
      width: 100%;
      height: 50vh;
      flex-direction: column;
    }

    .sidebar-inner {
      display: grid;
      grid-template-rows: 1fr auto 1fr;
      align-items: center;
      justify-items: center;
      height: 100%;
      padding: 1rem 1.5rem;
      text-align: center;
    }

    .sidebar-inner.active {
      grid-template-rows: 0.2fr auto 1fr;
    }

    .sidebar-flag {
      height: 80px;
      width: 80px;
    }

    .sidebar-title {
      font-size: 1.3rem;
    }

    .sidebar-avatar {
      width: 100px;
      height: 100px;
      border-width: 2px;
    }

    .sidebar-name {
      font-size: 1.6rem;
    }

    .sidebar-date {
      font-size: 1.1rem;
    }

    .sidebar-quote .quote-text {
      font-size: 2rem;
    }

    .sidebar-quote .quote-attr {
      font-size: 1.2rem;
    }

    .sidebar-act-slot {
      padding-top: 0.75rem;
    }

    .act-simple {
      font-size: 1.6rem;
    }

    .act-formal {
      font-size: 0.7rem;
    }

    .content {
      margin-left: 0;
      margin-top: 0;
    }


    .grid-viewport {
      top: 50vh;
      height: 50vh;
      padding: 0.5rem 0.75rem;
    }

    .grid-inner {
      display: flex;
      flex-direction: column;
      gap: 1px;
      width: 100%;
    }

    .g {
      width: 100%;
      height: 8px;
      border-radius: 2px;
    }

    .g.ev {
      height: 28px;
      border-radius: 4px;
      display: flex;
      align-items: center;
      justify-content: center;
      padding: 0 10px;
    }

    .g .g-label {
      display: block;
      font-size: 0.65rem;
      color: rgba(255, 255, 255, 0.7);
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }

    .g.sep {
      height: 28px;
      border-radius: 50%;
      width: 28px;
      margin: 6px 0;
      display: flex;
      align-items: center;
      justify-content: center;
      align-self: center;
      overflow: hidden;
      background: transparent;
    }

    .g.sep img {
      width: 28px;
      height: 28px;
      border-radius: 50%;
      object-fit: cover;
      object-position: top;
    }

    .g.ev:hover {
      transform: none;
    }
  }
</style>

<svelte:window bind:scrollY={y} bind:innerHeight={height} bind:innerWidth={innerWidth} />
{#if acts && govs}
  <main>
    <div class="main-inner">
      <aside
        class="sidebar"
        style={`background: ${activeIndex >= 0 ? parties[govs[currentPM].party] : '#2a2a2e'}`}>
        <div class="sidebar-inner" class:active={activeIndex >= 0}>
          {#if activeIndex >= 0 && govs[currentPM]}
            <div class="sidebar-hero">
              <span class="sidebar-name">{govs[currentPM].nickname}</span>
              <img
                class="sidebar-avatar"
                src={`/pms/${govs[currentPM].image}`}
                alt="pm" />
              <span class="sidebar-date">
                {(currentActDate || currentDate).split(' ')[0]}
                {' '}
                {(currentActDate || currentDate).split(' ')[1]}
              </span>
            </div>
            <div class="sidebar-act-slot">
              {#if currentAct && currentActName}
                <a class="sidebar-act" href={currentActLink} target="_blank">
                  <span class="act-simple">{currentAct}</span>
                  <span class="act-formal">{currentActName}</span>
                </a>
              {/if}
            </div>
          {:else}
            <div class="sidebar-hero">
              <img class="sidebar-flag" src="/favicon.png" alt="UK" />
              <div class="sidebar-quote">
                <span class="quote-text">"The UK is an <strong>elective dictatorship</strong>, absolute in theory, tolerable in practice"</span>
                <em class="quote-attr">Lord Hailsham, 1976</em>
              </div>
            </div>
          {/if}
        </div>
      </aside>

      <div class="content">
        <section
          class="grid-section"
          bind:this={gridSection}
          style={`height:${sectionHeight}px`}>
          <div class="grid-viewport">
            <div
              class="grid-inner"
              bind:this={gridInner}
              style={`transform: translateY(-${gridOffset}px)`}>
              {#each gridItems as item, idx}
                {#if item.isSeparator}
                  <div class="g sep">
                    <img src={`/pms/${item.pmImage}`} alt={item.pmName} />
                  </div>
                {:else}
                  <div
                    class="g"
                    class:lit={activeIndex >= 0 && idx <= activeIndex}
                    class:ev={item.isEvent}
                    class:active={activeIndex >= 0 && idx === activeIndex}
                    style={`--c:${item.partyColor}`}
                    on:click={item.isEvent ? () => scrollToItem(idx) : null}>
                    {#if item.isEvent && item.acts.length > 0}
                      <span class="g-label">{item.acts[0].Act}</span>
                    {/if}
                  </div>
                {/if}
              {/each}
            </div>
          </div>
        </section>
      </div>
    </div>
  </main>
{/if}
