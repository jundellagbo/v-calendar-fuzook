<script>
import Popover from './Popover';
import PopoverRow from './PopoverRow';
import Grid from './Grid';
import CalendarPane from './CalendarPane';
import CustomTransition from './CustomTransition';
import SvgIcon from './SvgIcon';
import AttributeStore from '@/utils/attributeStore';
import Attribute from '@/utils/attribute';
import {
  propOrDefaultMixin,
  rootMixin,
  safeScopedSlotMixin,
} from '@/utils/mixins';
import { addHorizontalSwipeHandler } from '@/utils/touch';
import DateInfo, { addDays, addMonths, addYears } from '@/utils/dateInfo';
import {
  pageForDate,
  pageForThisMonth,
  addPages,
  pageIsValid,
  pageIsEqualToPage,
  pageIsBeforePage,
  pageIsAfterPage,
  pageIsBetweenPages,
  createGuid,
  arrayHasItems,
  onSpaceOrEnter,
} from '@/utils/helpers';
import {
  isNumber,
  isDate,
  isArray,
  isObject,
  hasAny,
  omit,
  head,
  last,
} from '@/utils/_';

export default {
  name: 'VCalendar',
  render(h) {
    // Renderer for calendar panes
    const panes = this.pages.map((page, i) =>
      h(CalendarPane, {
        attrs: {
          ...this.$attrs,
          attributes: this.store,
        },
        props: {
          titlePosition: this.titlePosition_,
          page,
          minPage: this.minPage_,
          maxPage: this.maxPage_,
          canMove: this.canMove,
        },
        on: {
          ...this.$listeners,
          'update:page': e => this.refreshPages({ page: e, position: i + 1 }),
          dayfocusin: e => {
            this.lastFocusedDay = e;
            this.$emit('dayfocusin', e);
          },
          dayfocusout: e => {
            this.lastFocusedDay = null;
            this.$emit('dayfocusout', e);
          },
        },
        scopedSlots: this.$scopedSlots,
        key: page.key,
        ref: 'pages',
        refInFor: true,
      }),
    );

    // Renderer for calendar arrows
    const getArrowButton = isPrev => {
      const click = () => (isPrev ? this.movePrev() : this.moveNext());
      const keydown = e => onSpaceOrEnter(e, click);
      const isDisabled = isPrev ? !this.canMovePrev : !this.canMoveNext;
      return h(
        'div',
        {
          class: [
            `vc-flex vc-justify-center vc-items-center vc-cursor-pointer vc-select-none ${
              isDisabled
                ? 'vc-opacity-25 vc-pointer-events-none vc-cursor-not-allowed'
                : 'vc-pointer-events-auto'
            }`,
            this.theme_.arrows,
          ],
          attrs: {
            role: 'button',
            tabindex: '0',
          },
          on: {
            click,
            keydown,
          },
        },
        [
          (isPrev
            ? this.safeScopedSlot('header-left-button', { click })
            : this.safeScopedSlot('header-right-button', { click })) ||
            h(SvgIcon, {
              props: {
                name: isPrev ? 'left-arrow' : 'right-arrow',
              },
            }),
        ],
      );
    };

    // Renderer for popover
    const getDayPopover = () =>
      h(Popover, {
        props: {
          id: this.sharedState.dayPopoverId,
          contentClass: this.theme_.dayPopoverContainer,
        },
        scopedSlots: {
          default: ({ args: day, updateLayout, hide }) => {
            const attributes = Object.values(day.attributes).filter(
              a => a.popover,
            );
            const masks = this.locale_.masks;
            const format = this.format;
            const dayTitle = format(day.date, masks.dayPopover);
            return (
              this.safeScopedSlot('day-popover', {
                day,
                attributes,
                masks,
                format,
                dayTitle,
                updateLayout,
                hide,
              }) ||
              h('div', [
                // Show popover header only if format is defined
                masks.dayPopover &&
                  h(
                    'div',
                    {
                      class: ['vc-text-center', this.theme_.dayPopoverHeader],
                    },
                    [dayTitle],
                  ),
                attributes.map(attribute =>
                  h(PopoverRow, {
                    key: attribute.key,
                    props: {
                      attribute,
                    },
                  }),
                ),
              ])
            );
          },
        },
      });

    // Renderer for calendar container
    const getContainerGrid = () =>
      h(
        'div',
        {
          attrs: {
            'data-helptext':
              'Press the arrow keys to navigate by day, Home and End to navigate to week ends, PageUp and PageDown to navigate by month, Alt+PageUp and Alt+PageDown to navigate by year',
          },
          class: [
            'vc-container',
            'vc-relative',
            'vc-reset',
            {
              'vc-min-w-full': this.isExpanded,
            },
            this.theme_.container,
          ],
          on: {
            keydown: this.handleKeydown,
          },
          ref: 'container',
        },
        [
          h(
            'div',
            {
              class: [
                'vc-w-full vc-relative',
                { 'vc-overflow-hidden': this.inTransition },
              ],
            },
            [
              h(
                CustomTransition,
                {
                  props: {
                    name: this.transitionName,
                  },
                  on: {
                    beforeEnter: () => {
                      this.inTransition = true;
                    },
                    afterEnter: () => {
                      this.inTransition = false;
                    },
                  },
                },
                [
                  h(
                    Grid,
                    {
                      class: 'grid',
                      props: {
                        rows: this.rows,
                        columns: this.columns,
                        columnWidth: 'minmax(256px, 1fr)',
                        disableFocus: true,
                      },
                      attrs: {
                        ...this.$attrs,
                      },
                      key: arrayHasItems(this.pages) ? this.pages[0].key : '',
                    },
                    panes,
                  ),
                ],
              ),
              h(
                'div',
                {
                  class: [`vc-arrows-container title-${this.titlePosition_}`],
                },
                [getArrowButton(true), getArrowButton(false)],
              ),
            ],
          ),
          getDayPopover(),
        ],
      );

    return getContainerGrid();
  },
  mixins: [propOrDefaultMixin, rootMixin, safeScopedSlotMixin],
  provide() {
    return {
      sharedState: this.sharedState,
    };
  },
  props: {
    rows: {
      type: Number,
      default: 1,
    },
    columns: {
      type: Number,
      default: 1,
    },
    step: Number,
    titlePosition: String,
    isExpanded: Boolean,
    fromDate: Date,
    toDate: Date,
    fromPage: Object,
    toPage: Object,
    minDate: null,
    maxDate: null,
    minPage: Object,
    maxPage: Object,
    disabledDates: null,
    availableDates: null,
    transition: String,
    attributes: [Object, Array],
    disablePageSwipe: Boolean,
  },
  data() {
    return {
      pages: [],
      store: null,
      lastFocusedDay: null,
      focusableDay: new Date().getDate(),
      transitionName: '',
      inTransition: false,
      sharedState: {
        dayPopoverId: createGuid(),
        theme: {},
        masks: {},
        locale: {},
      },
    };
  },
  computed: {
    titlePosition_() {
      return this.propOrDefault('titlePosition', 'titlePosition');
    },
    minPage_() {
      return this.minPage || pageForDate(this.locale_.toDate(this.minDate));
    },
    maxPage_() {
      return this.maxPage || pageForDate(this.locale_.toDate(this.maxDate));
    },
    count() {
      return this.rows * this.columns;
    },
    step_() {
      return this.step || this.count;
    },
    canMovePrev() {
      return (
        !pageIsValid(this.minPage_) ||
        pageIsAfterPage(this.pages[0], this.minPage_)
      );
    },
    canMoveNext() {
      return (
        !pageIsValid(this.maxPage_) ||
        pageIsBeforePage(this.pages[this.pages.length - 1], this.maxPage_)
      );
    },
    disabledAttribute() {
      // Build up a complete list of disabled dates
      let dates = [];
      // Initialize with disabled dates prop, if any
      if (this.disabledDates) {
        dates = isArray(this.disabledDates)
          ? this.disabledDates
          : [this.disabledDates];
      }
      // Add disabled dates for minDate and maxDate props
      const minDate = this.locale_.toDate(this.minDate);
      const maxDate = this.locale_.toDate(this.maxDate);
      if (minDate) {
        dates.push({ start: null, end: addDays(minDate, -1) });
      }
      if (maxDate) {
        dates.push({ start: addDays(maxDate, 1), end: null });
      }
      // Return the new disabled attribute
      return new Attribute(
        {
          key: 'disabled',
          dates,
          excludeDates: this.availableDates,
          excludeMode: 'includes',
          order: 100,
        },
        this.theme_,
        this.locale_,
      );
    },
  },
  watch: {
    locale_() {
      this.refreshLocale();
      this.refreshPages({ page: head(this.pages), ignoreCache: true });
      this.initStore();
    },
    theme_() {
      this.refreshTheme();
      this.initStore();
    },
    fromPage(val) {
      const firstPage = this.pages && this.pages[0];
      if (pageIsEqualToPage(val, firstPage)) return;
      this.refreshPages();
    },
    toPage(val) {
      const lastPage = this.pages && this.pages[this.pages.length - 1];
      if (pageIsEqualToPage(val, lastPage)) return;
      this.refreshPages();
    },
    count() {
      this.refreshPages();
    },
    attributes(val) {
      const { adds, deletes } = this.store.refresh(val);
      this.refreshAttrs(this.pages, adds, deletes);
    },
    pages(val) {
      this.refreshAttrs(val, this.store.list, null, true);
    },
    disabledAttribute() {
      this.refreshDisabledDays();
    },
    lastFocusedDay(val) {
      if (val) {
        this.focusableDay = val.day;
        this.refreshFocusableDays();
      }
    },
  },
  created() {
    this.refreshLocale();
    this.refreshTheme();
    this.initStore();
    this.refreshPages();
  },
  mounted() {
    if (!this.disablePageSwipe) {
      // Add swipe handler to move to next and previous pages
      const removeHandlers = addHorizontalSwipeHandler(
        this.$refs.container,
        ({ toLeft, toRight }) => {
          if (toLeft) {
            this.moveNext();
          } else if (toRight) {
            this.movePrev();
          }
        },
        this.$vc.defaults.touch,
      );
      // Clean up on destroy
      this.$once('beforeDestroy', () => removeHandlers());
    }
  },
  methods: {
    refreshLocale() {
      this.sharedState.locale = this.locale_;
      this.sharedState.masks = this.locale_.masks;
    },
    refreshTheme() {
      this.sharedState.theme = this.theme_;
    },
    canMove(page) {
      return pageIsBetweenPages(page, this.minPage_, this.maxPage_);
    },
    movePrev() {
      this.move(-this.step_);
    },
    moveNext() {
      this.move(this.step_);
    },
    move(monthsOrPage, { completion = null, position = 1 } = {}) {
      const page = isNumber(monthsOrPage)
        ? addPages(this.pages[0], monthsOrPage)
        : monthsOrPage;
      this.refreshPages({
        page,
        position,
      });
      if (completion) {
        this.$nextTick(() => completion());
      }
    },
    refreshPages({ page, position = 1, ignoreCache } = {}) {
      // Calculate the page to start displaying from
      let fromPage = null;
      // 1. Try the page parameter
      if (pageIsValid(page)) {
        const pagesToAdd =
          position > 0 ? 1 - position : -(this.count + position);
        fromPage = addPages(page, pagesToAdd);
      } else {
        // 2. Try the fromPage prop
        fromPage =
          this.fromPage || pageForDate(this.locale_.toDate(this.fromDate));
        if (!pageIsValid(fromPage)) {
          // 3. Try the toPage prop
          const toPage =
            this.toPage || pageForDate(this.locale_.toDate(this.toPage));
          if (pageIsValid(toPage)) {
            fromPage = addPages(toPage, 1 - this.count);
          } else {
            // 4. Try the first attribute
            fromPage = this.getPageForAttributes();
          }
        }
      }
      // 5. Fall back to today's page
      fromPage = pageIsValid(fromPage) ? fromPage : pageForThisMonth();
      // Adjust from page within allowed min/max pages
      const toPage = addPages(fromPage, this.count - 1);
      if (pageIsBeforePage(fromPage, this.minPage_)) {
        fromPage = this.minPage_;
      } else if (pageIsAfterPage(toPage, this.maxPage_)) {
        fromPage = addPages(this.maxPage_, 1 - this.count);
      }
      // Create the new pages
      const pages = [...Array(this.count).keys()].map(i =>
        this.buildPage(addPages(fromPage, i), ignoreCache),
      );
      // Refresh disabled days for new pages
      this.refreshDisabledDays(pages);
      // Refresh focusable days for new pages
      this.refreshFocusableDays(pages);
      // Assign the new transition
      this.transitionName = this.getPageTransition(this.pages[0], pages[0]);
      // Assign the new pages
      this.pages = pages;
      // Emit page update events
      this.$emit('update:from-page', fromPage);
      this.$emit('update:to-page', toPage);
    },
    refreshDisabledDays(pages) {
      this.getPageDays(pages).forEach(
        d =>
          (d.isDisabled =
            !!this.disabledAttribute && this.disabledAttribute.includesDay(d)),
      );
    },
    refreshFocusableDays(pages) {
      this.getPageDays(pages).forEach(
        d => (d.isFocusable = d.inMonth && d.day === this.focusableDay),
      );
    },
    getPageDays(pages = this.pages) {
      return pages.reduce((prev, curr) => prev.concat(curr.days), []);
    },
    getPageTransition(oldPage, newPage) {
      if (this.transition === 'none') return this.transition;
      if (
        this.transition === 'fade' ||
        (!this.transition && this.count > 1) ||
        !pageIsValid(oldPage) ||
        !pageIsValid(newPage)
      ) {
        return 'fade';
      }
      // Moving to a previous page
      const movePrev = pageIsBeforePage(newPage, oldPage);
      // Vertical slide
      if (this.transition === 'slide-v') {
        return movePrev ? 'slide-down' : 'slide-up';
      }
      // Horizontal slide
      return movePrev ? 'slide-right' : 'slide-left';
    },
    getPageForAttributes() {
      let page = null;
      const attr = this.store.pinAttr;
      if (attr && attr.hasDates) {
        let [date] = attr.dates;
        date = date.start || date.date;
        page = pageForDate(this.locale_.toDate(date));
      }
      return page;
    },
    buildPage({ month, year }, ignoreCache) {
      const key = `${year.toString()}-${month.toString()}`;
      let page = this.pages.find(p => p.key === key);
      if (!page || ignoreCache) {
        const date = new Date(year, month - 1, 15);
        const monthComps = this.locale_.getMonthComps(month, year);
        const prevMonthComps = this.locale_.getPrevMonthComps(month, year);
        const nextMonthComps = this.locale_.getNextMonthComps(month, year);
        page = {
          key,
          month,
          year,
          title: this.locale_.format(date, this.locale_.masks.title),
          shortMonthLabel: this.locale_.format(date, 'MMM'),
          monthLabel: this.locale_.format(date, 'MMMM'),
          shortYearLabel: year.toString().substring(2),
          yearLabel: year.toString(),
          monthComps,
          prevMonthComps,
          nextMonthComps,
          canMove: pg => this.canMove(pg),
          move: pg => this.move(pg),
          moveThisMonth: () => this.moveThisMonth(),
          movePrevMonth: () => this.move(prevMonthComps),
          moveNextMonth: () => this.move(nextMonthComps),
          refresh: true,
        };
        // Assign day info
        page.days = this.locale_.getCalendarDays(page);
      }
      return page;
    },
    initStore() {
      // Create a new attribute store
      this.store = new AttributeStore(
        this.theme_,
        this.locale_,
        this.attributes,
      );
      // Refresh attributes for existing pages
      this.refreshAttrs(this.pages, this.store.list, [], true);
    },
    refreshAttrs(pages = [], adds = [], deletes = [], reset) {
      if (!arrayHasItems(pages)) return;
      // For each page...
      pages.forEach(p => {
        // For each day...
        p.days.forEach(d => {
          let map = {};
          // If resetting...
          if (reset) {
            // Flag day for refresh if it has attributes
            d.refresh = arrayHasItems(d.attributes);
          } else if (hasAny(d.attributesMap, deletes)) {
            // Delete attributes from the delete list
            map = omit(d.attributesMap, deletes);
            // Flag day for refresh
            d.refresh = true;
          } else {
            // Get the existing attributes
            map = d.attributesMap || {};
          }
          // For each attribute to add...
          adds.forEach(attr => {
            // Add it if it includes the current day
            const targetDate = attr.includesDay(d);
            if (targetDate) {
              const newAttr = {
                ...attr,
                targetDate,
              };
              map[attr.key] = newAttr;
              // Flag day for refresh
              d.refresh = true;
            }
          });
          // Reassign day attributes
          if (d.refresh) {
            d.attributesMap = map;
          }
        });
      });
      // Refresh pages
      this.$nextTick(() => {
        this.$refs.pages.forEach(p => p.refresh());
      });
    },
    showPageRange(range) {
      let fromPage;
      let toPage;
      if (isDate(range)) {
        fromPage = pageForDate(range);
      } else if (isObject(range)) {
        const { month, year } = range;
        const { from, to } = range;
        if (isNumber(month) && isNumber(year)) {
          fromPage = range;
        } else if (from || to) {
          fromPage = isDate(from) ? pageForDate(from) : from;
          toPage = isDate(to) ? pageForDate(to) : to;
        }
      } else {
        return;
      }
      const lastPage = last(this.pages);
      let page = fromPage;
      if (pageIsAfterPage(toPage, lastPage)) {
        page = addPages(toPage, -(this.pages.length - 1));
      }
      if (pageIsBeforePage(fromPage, page)) {
        page = fromPage;
      }
      this.refreshPages({ page });
    },
    handleKeydown(e) {
      const day = this.lastFocusedDay;
      if (day != null) {
        day.event = e;
        this.handleDayKeydown(day);
      }
    },
    handleDayKeydown(day) {
      const { date, event } = day;
      let newDate = null;
      switch (event.key) {
        case 'ArrowLeft': {
          // Move to previous day
          newDate = addDays(date, -1);
          break;
        }
        case 'ArrowRight': {
          // Move to next day
          newDate = addDays(date, 1);
          break;
        }
        case 'ArrowUp': {
          // Move to previous week
          newDate = addDays(date, -7);
          break;
        }
        case 'ArrowDown': {
          // Move to next week
          newDate = addDays(date, 7);
          break;
        }
        case 'Home': {
          // Move to first weekday position
          newDate = addDays(date, -day.weekdayPosition + 1);
          break;
        }
        case 'End': {
          // Move to last weekday position
          newDate = addDays(date, day.weekdayPositionFromEnd);
          break;
        }
        case 'PageUp': {
          if (event.altKey) {
            // Move to previous year w/ Alt/Option key
            newDate = addYears(date, -1);
          } else {
            // Move to previous month
            newDate = addMonths(date, -1);
          }
          break;
        }
        case 'PageDown': {
          if (event.altKey) {
            // Move to next year w/ Alt/Option key
            newDate = addYears(date, 1);
          } else {
            // Move to next month
            newDate = addMonths(date, 1);
          }
          break;
        }
      }
      if (newDate) {
        event.preventDefault();
        this.setFocusedDate(newDate);
      }
    },
    setFocusedDate(date) {
      if (!date || !arrayHasItems(this.pages)) return;
      const dateInfo = new DateInfo(date, { locale: this.locale_ });

      const day = this.getPageDays().find(
        d => d.inMonth && dateInfo.includesDay(d),
      );
      if (day) {
        // Set focus on the element for the date
        const focusableEl = this.$el.querySelector(
          `.id-${day.id}.in-month .vc-focusable`,
        );
        if (focusableEl) {
          focusableEl.focus();
        }
      } else {
        const page = pageForDate(date);
        const position = pageIsBeforePage(page, this.pages[0]) ? -1 : 1;
        this.move(page, {
          completion: () => this.setFocusedDate(date),
          position,
        });
      }
    },
  },
};
</script>

<style lang="postcss">
.vc-container {
  --slide-translate: 22px;
  --slide-duration: 0.15s;
  --slide-timing: ease;

  --header-padding: 10px 10px 0 10px;
  --title-padding: 0 8px;
  --arrows-padding: 10px;
  --arrow-font-size: 26px;
  --weekday-padding: 5px 0;
  --weeks-padding: 5px 6px 7px 6px;
  --arrow-horizontal-offset: 10px;
  --arrow-vertical-offset: 11px;

  --nav-container-width: 170px;

  --day-min-height: 28px;
  --day-content-width: 28px;
  --day-content-height: 28px;
  --day-content-margin: 1.6px auto;
  --day-content-transition-time: 0.13s ease-in;
  --day-content-bg-color-hover: hsla(211, 25%, 84%, 0.3);
  --day-content-dark-bg-color-hover: hsla(216, 15%, 52%, 0.3);
  --day-content-bg-color-focus: hsla(211, 25%, 84%, 0.4);
  --day-content-dark-bg-color-focus: hsla(216, 15%, 52%, 0.4);

  --highlight-height: 28px;

  --dot-diameter: 5px;
  --dot-border-radius: 50%;
  --dot-spacing: 3px;

  --bar-height: 3px;
  --bars-width: 75%;

  font-family: BlinkMacSystemFont, -apple-system, 'Segoe UI', 'Roboto', 'Oxygen',
    'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue',
    'Helvetica', 'Arial', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  width: max-content;
}

.vc-arrows-container {
  width: 100%;
  position: absolute;
  top: 0;
  display: flex;
  justify-content: space-between;
  padding: var(--arrows-padding);
  pointer-events: none;
  &.title-left {
    justify-content: flex-end;
  }
  &.title-right {
    justify-content: flex-start;
  }
}
</style>
