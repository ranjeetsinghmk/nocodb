<script lang="ts" setup>
import dayjs from 'dayjs'
import type { ColumnType } from 'nocodb-sdk'
import type { Row } from '~/lib/types'

const emits = defineEmits(['expandRecord', 'newRecord'])

const {
  selectedDateRange,
  formattedData,
  formattedSideBarData,
  calendarRange,
  displayField,
  selectedTime,
  updateRowProperty,
  sideBarFilterOption,
  showSideMenu,
} = useCalendarViewStoreOrThrow()

const { $e } = useNuxtApp()

const container = ref<null | HTMLElement>(null)

const scrollContainer = ref<null | HTMLElement>(null)

const { width: containerWidth } = useElementSize(container)

const { isUIAllowed } = useRoles()

const meta = inject(MetaInj, ref())

const fields = inject(FieldsInj, ref())

const { fields: _fields } = useViewColumnsOrThrow()

const fieldStyles = computed(() => {
  if (!_fields.value) return new Map()
  return new Map(
    _fields.value.map((field) => [
      field.fk_column_id,
      {
        underline: field.underline,
        bold: field.bold,
        italic: field.italic,
      },
    ]),
  )
})

const getFieldStyle = (field: ColumnType) => {
  return fieldStyles.value.get(field.id)
}
const calculateNewDates = useMemoize(
  ({
    startDate,
    endDate,
    scheduleStart,
    scheduleEnd,
  }: {
    startDate: dayjs.Dayjs
    endDate: dayjs.Dayjs
    scheduleStart: dayjs.Dayjs
    scheduleEnd: dayjs.Dayjs
  }) => {
    // If the end date is not valid, we set it to 15 minutes after the start date
    if (!endDate?.isValid()) {
      endDate = startDate.clone().add(15, 'minutes')
    }

    // If the start date is before the start of the schedule, we set it to the start of the schedule
    // If the end date is after the end of the schedule, we set it to the end of the schedule
    // This is to ensure that the records are within the bounds of the schedule and do not overflow
    if (startDate.isBefore(scheduleStart, 'minutes')) {
      startDate = scheduleStart.clone()
    }
    if (endDate.isAfter(scheduleEnd, 'minutes')) {
      endDate = scheduleEnd.clone()
    }

    return { startDate, endDate }
  },
)

// Since it is a datetime Week view, we need to create a 2D array of dayjs objects to represent the hours in a day for each day in the week
const datesHours = computed(() => {
  const datesHours: Array<Array<dayjs.Dayjs>> = []
  let startOfWeek = dayjs(selectedDateRange.value.start) ?? dayjs().startOf('week')
  const endOfWeek = dayjs(selectedDateRange.value.end) ?? dayjs().endOf('week')

  while (startOfWeek.isSameOrBefore(endOfWeek)) {
    const hours: Array<dayjs.Dayjs> = []
    for (let i = 0; i < 24; i++) {
      hours.push(
        dayjs()
          .hour(i)
          .minute(0)
          .second(0)
          .millisecond(0)
          .year(startOfWeek.year())
          .month(startOfWeek.month())
          .date(startOfWeek.date()),
      )
    }
    datesHours.push(hours)
    startOfWeek = startOfWeek.add(1, 'day')
  }
  return datesHours
})

const getDayIndex = (date: dayjs.Dayjs) => {
  let dayIndex = date.day() - 1
  if (dayIndex === -1) {
    dayIndex = 6
  }
  return dayIndex
}

const getGridTime = (date: dayjs.Dayjs, round = false) => {
  const gridCalc = date.hour() * 60 + date.minute()
  if (round) {
    return Math.ceil(gridCalc)
  } else {
    return Math.floor(gridCalc)
  }
}

const getGridTimeSlots = (from: dayjs.Dayjs, to: dayjs.Dayjs) => {
  return {
    from: getGridTime(from, false),
    to: getGridTime(to, true) - 1,
    dayIndex: getDayIndex(from),
  }
}

const hasSlotForRecord = (
  columnArray: Row[],
  dates: {
    fromDate: dayjs.Dayjs
    toDate: dayjs.Dayjs
  },
) => {
  const { fromDate, toDate } = dates

  if (!fromDate || !toDate) return false

  for (const column of columnArray) {
    const columnFromCol = column.rowMeta.range?.fk_from_col
    const columnToCol = column.rowMeta.range?.fk_to_col

    if (!columnFromCol) return false

    const { startDate: columnFromDate, endDate: columnToDate } = calculateNewDates({
      startDate: dayjs(column.row[columnFromCol.title!]),
      endDate: columnToCol
        ? dayjs(column.row[columnToCol.title!])
        : dayjs(column.row[columnFromCol.title!]).add(1, 'hour').subtract(1, 'minute'),
      scheduleStart: dayjs(selectedDateRange.value.start).startOf('day'),
      scheduleEnd: dayjs(selectedDateRange.value.end).endOf('day'),
    })

    if (
      fromDate.isBetween(columnFromDate, columnToDate, null, '[]') ||
      toDate.isBetween(columnFromDate, columnToDate, null, '[]')
    ) {
      return false
    }
  }
  return true
}

const getMaxOverlaps = ({
  row,
  columnArray,
  graph,
}: {
  row: Row
  columnArray: Array<Array<Array<Row>>>
  graph: Map<string, Set<string>>
}) => {
  const id = row.rowMeta.id as string

  const visited: Set<string> = new Set()

  const dayIndex = row.rowMeta.dayIndex
  const overlapIndex = columnArray[dayIndex].findIndex((column) => column.findIndex((r) => r.rowMeta.id === id) !== -1) + 1
  const dfs = (id: string): number => {
    visited.add(id)
    let maxOverlaps = 1
    const neighbors = graph.get(id)
    if (neighbors) {
      for (const neighbor of neighbors) {
        if (maxOverlaps >= columnArray[dayIndex].length) return maxOverlaps
        if (!visited.has(neighbor)) {
          maxOverlaps = Math.min(Math.max(maxOverlaps, dfs(neighbor) + 1), columnArray[dayIndex].length)
        }
      }
    }

    return maxOverlaps
  }

  let maxOverlaps = 1
  if (graph.has(id)) {
    maxOverlaps = dfs(id)
  }
  return { maxOverlaps, dayIndex, overlapIndex }
}

const recordsAcrossAllRange = computed<{
  records: Array<Row>
  gridTimeMap: Map<
    number,
    Map<
      number,
      {
        count: number
        id: string[]
      }
    >
  >
}>(() => {
  if (!formattedData.value || !calendarRange.value || !container.value || !scrollContainer.value)
    return {
      records: [],
      gridTimeMap: new Map(),
    }
  const perWidth = containerWidth.value / 7
  const perHeight = 52

  const scheduleStart = dayjs(selectedDateRange.value.start).startOf('day')
  const scheduleEnd = dayjs(selectedDateRange.value.end).endOf('day')

  const columnArray: Array<Array<Array<Row>>> = [[[]]]
  const gridTimeMap = new Map<
    number,
    Map<
      number,
      {
        count: number
        id: string[]
      }
    >
  >()
  const recordsToDisplay: Array<Row> = []

  calendarRange.value.forEach((range) => {
    const fromCol = range.fk_from_col
    const toCol = range.fk_to_col

    // We fetch all the records that match the calendar ranges in a single time.
    // But not all fetched records are valid for the certain range, so we filter them out & sort them
    const sortedFormattedData = [...formattedData.value]
      .filter((record) => {
        const fromDate = record.row[fromCol!.title!] ? dayjs(record.row[fromCol!.title!]) : null

        if (fromCol && toCol) {
          const fromDate = record.row[fromCol.title!] ? dayjs(record.row[fromCol.title!]) : null
          const toDate = record.row[toCol.title!] ? dayjs(record.row[toCol.title!]) : null

          return fromDate && toDate && !toDate.isBefore(fromDate)
        } else if (fromCol && !toCol) {
          return !!fromDate
        }
        return false
      })
      .sort((a, b) => {
        const aDate = dayjs(a.row[fromCol!.title!])
        const bDate = dayjs(b.row[fromCol!.title!])
        return aDate.isBefore(bDate) ? 1 : -1
      })

    for (const record of sortedFormattedData) {
      const id = record.rowMeta.id ?? generateRandomNumber()

      if (fromCol && toCol) {
        const { startDate, endDate } = calculateNewDates({
          startDate: dayjs(record.row[fromCol.title!]),
          endDate: dayjs(record.row[toCol.title!]),
          scheduleStart,
          scheduleEnd,
        })

        // Setting the current start date to the start date of the record
        let currentStartDate: dayjs.Dayjs = startDate.clone()

        // We loop through the start date to the end date and create a record for each day as it spans bottom to top
        while (currentStartDate.isSameOrBefore(endDate!, 'day')) {
          const currentEndDate = currentStartDate.clone().endOf('day')
          const recordStart: dayjs.Dayjs = currentEndDate.isSame(startDate, 'day') ? startDate : currentStartDate
          const recordEnd = currentEndDate.isSame(endDate, 'day') ? endDate : currentEndDate

          const dayIndex = getDayIndex(recordStart)

          // We calculate the index of the start and end hour in the day
          const startHourIndex = Math.max(
            (datesHours.value[dayIndex] ?? []).findIndex((h) => h.format('HH:mm') === recordStart.format('HH:mm')),
            0,
          )
          const endHourIndex = Math.max(
            (datesHours.value[dayIndex] ?? []).findIndex((h) => h.format('HH:mm') === recordEnd?.startOf('hour').format('HH:mm')),
            0,
          )

          let position: 'topRounded' | 'bottomRounded' | 'rounded' | 'none' = 'rounded'
          const isSelectedDay = (date: dayjs.Dayjs) => date.isSame(currentStartDate, 'day')
          const isBeforeSelectedDay = (date: dayjs.Dayjs) => date.isBefore(currentStartDate, 'day')
          const isAfterSelectedDay = (date: dayjs.Dayjs) => date.isAfter(currentStartDate, 'day')

          if (isSelectedDay(startDate) && isSelectedDay(endDate)) {
            position = 'rounded'
          } else if (isBeforeSelectedDay(startDate) && isAfterSelectedDay(endDate)) {
            position = 'none'
          } else if (isSelectedDay(startDate) && isAfterSelectedDay(endDate)) {
            position = 'topRounded'
          } else if (isBeforeSelectedDay(startDate) && isSelectedDay(endDate)) {
            position = 'bottomRounded'
          } else {
            position = 'none'
          }

          let style: Partial<CSSStyleDeclaration> = {}

          const spanHours = endHourIndex - startHourIndex + 1

          const top = startHourIndex * perHeight

          const height = (endHourIndex - startHourIndex + 1) * perHeight - spanHours - 5

          style = {
            ...style,
            top: `${top}px`,
            height: `${height}px`,
          }

          recordsToDisplay.push({
            ...record,
            rowMeta: {
              ...record.rowMeta,
              id,
              style,
              range,
              position,
              dayIndex,
            },
          })
          // We set the current start date to the next day
          currentStartDate = currentStartDate.add(1, 'day').hour(0).minute(0)
        }
      } else if (fromCol) {
        // If there is no toColumn chosen in the range
        const { startDate } = calculateNewDates({
          startDate: dayjs(record.row[fromCol.title!]),
          endDate: dayjs(record.row[fromCol.title!]).add(1, 'hour').subtract(1, 'minute'),
          scheduleStart,
          scheduleEnd,
        })

        let style: Partial<CSSStyleDeclaration> = {}

        const dayIndex = getDayIndex(startDate)

        const minutes = (startDate.minute() / 60 + startDate.hour()) * perHeight

        style = {
          ...style,
          top: `${minutes + 1}px`,
          height: `${perHeight - 2}px`,
        }

        recordsToDisplay.push({
          ...record,
          rowMeta: {
            ...record.rowMeta,
            id,
            position: 'rounded',
            style,
            range,
            dayIndex,
          },
        })
      }
    }

    recordsToDisplay.sort((a, b) => {
      const fromColA = a.rowMeta.range?.fk_from_col
      const fromColB = b.rowMeta.range?.fk_from_col
      if (!fromColA || !fromColB) return 0
      return dayjs(a.row[fromColA.title!]).isBefore(dayjs(b.row[fromColB.title!])) ? -1 : 1
    })

    for (const record of recordsToDisplay) {
      const fromCol = record.rowMeta.range?.fk_from_col
      const toCol = record.rowMeta.range?.fk_to_col

      if (!fromCol) continue
      const { startDate, endDate } = calculateNewDates({
        startDate: dayjs(record.row[fromCol.title!]),
        endDate: toCol ? dayjs(record.row[toCol.title!]) : dayjs(record.row[fromCol.title!]).add(1, 'hour').subtract(1, 'minute'),
        scheduleStart,
        scheduleEnd,
      })

      const gridTimes = getGridTimeSlots(startDate, endDate)

      const dayIndex = record.rowMeta.dayIndex ?? gridTimes.dayIndex

      for (let gridCounter = gridTimes.from; gridCounter <= gridTimes.to; gridCounter++) {
        if (!gridTimeMap.has(dayIndex)) {
          gridTimeMap.set(
            dayIndex,
            new Map<
              number,
              {
                count: number
                id: string[]
              }
            >(),
          )
        }

        if (!gridTimeMap.get(dayIndex)?.has(gridCounter)) {
          gridTimeMap.set(dayIndex, (gridTimeMap.get(dayIndex) ?? new Map()).set(gridCounter, { count: 0, id: [] }))
        }

        const idArray = gridTimeMap.get(dayIndex)!.get(gridCounter)!.id
        idArray.push(record.rowMeta.id!)
        const count = gridTimeMap.get(dayIndex)!.get(gridCounter)!.count + 1

        gridTimeMap.set(dayIndex, (gridTimeMap.get(dayIndex) ?? new Map()).set(gridCounter, { count, id: idArray }))
      }

      let foundAColumn = false

      if (!columnArray[dayIndex]) {
        columnArray[dayIndex] = []
      }

      for (const column in columnArray[dayIndex]) {
        if (hasSlotForRecord(columnArray[dayIndex][column], { fromDate: startDate, toDate: endDate })) {
          columnArray[dayIndex][column].push(record)
          foundAColumn = true
          break
        }
      }

      if (!foundAColumn) {
        columnArray[dayIndex].push([record])
      }
    }

    const graph: Map<number, Map<string, Set<string>>> = new Map()

    for (const dayIndex of gridTimeMap.keys()) {
      if (!graph.has(dayIndex)) {
        graph.set(dayIndex, new Map())
      }
      for (const [_gridTime, { id: ids }] of gridTimeMap.get(dayIndex)) {
        for (const id1 of ids) {
          if (!graph.get(dayIndex).has(id1)) {
            graph.get(dayIndex).set(id1, new Set())
          }
          for (const id2 of ids) {
            if (id1 !== id2) {
              if (!graph.get(dayIndex).get(id1).has(id2)) {
                graph.get(dayIndex).get(id1).add(id2)
              }
            }
          }
        }
      }
    }

    for (const dayIndex in columnArray) {
      for (const columnIndex in columnArray[dayIndex]) {
        for (const record of columnArray[dayIndex][columnIndex]) {
          record.rowMeta.overLapIteration = parseInt(columnIndex) + 1
        }
      }
    }
    for (const record of recordsToDisplay) {
      const {
        maxOverlaps,
        overlapIndex,
        dayIndex: tDayIndex,
      } = getMaxOverlaps({
        row: record,
        columnArray,
        graph: graph.get(record.rowMeta.dayIndex!) ?? new Map(),
      })

      const dayIndex = record.rowMeta.dayIndex ?? tDayIndex
      record.rowMeta.numberOfOverlaps = maxOverlaps

      let width = 0
      let left = 100
      const majorLeft = dayIndex * perWidth
      let display = 'block'

      if (record.rowMeta.overLapIteration! - 1 > 2) {
        display = 'none'
      } else {
        width = 100 / Math.min(maxOverlaps, 3) / 7
        left = width * (overlapIndex - 1)
      }
      record.rowMeta.style = {
        ...record.rowMeta.style,
        left: `calc(${majorLeft}px + ${left}%)`,
        width: `calc(${width}%)`,
        display,
      }
    }
  })

  return {
    records: recordsToDisplay,
    gridTimeMap,
  }
})

const resizeInProgress = ref(false)

const dragTimeout = ref<ReturnType<typeof setTimeout>>()

const hoverRecord = ref<string | null>()

const resizeDirection = ref<'right' | 'left' | null>()

const resizeRecord = ref<Row | null>()

const isDragging = ref(false)

const dragRecord = ref<Row>()

const useDebouncedRowUpdate = useDebounceFn((row: Row, updateProperty: string[], isDelete: boolean) => {
  updateRowProperty(row, updateProperty, isDelete)
}, 500)

const onResize = (event: MouseEvent) => {
  if (!isUIAllowed('dataEdit') || !container.value || !resizeRecord.value || !scrollContainer.value) return

  const { width, left, top, bottom } = container.value.getBoundingClientRect()

  const { scrollHeight } = container.value

  // If the mouse is near the bottom of the container, we scroll down
  // If the mouse is near the top of the container, we scroll up
  if (event.clientY > bottom - 20) {
    container.value.scrollTop += 10
  } else if (event.clientY < top + 20) {
    container.value.scrollTop -= 10
  }

  // We calculate the percentage of the mouse position in the container
  // percentX is used for the day and percentY is used for the hour
  const percentX = (event.clientX - left - window.scrollX) / width
  const percentY = (event.clientY - top + container.value.scrollTop) / scrollHeight

  const fromCol = resizeRecord.value.rowMeta.range?.fk_from_col
  const toCol = resizeRecord.value.rowMeta.range?.fk_to_col

  if (!fromCol || !toCol) return

  const ogEndDate = dayjs(resizeRecord.value.row[toCol.title!])
  const ogStartDate = dayjs(resizeRecord.value.row[fromCol.title!])

  const day = Math.floor(percentX * 7)
  const hour = Math.floor(percentY * 23)
  const minutes = Math.round((percentY * 24 * 60) % 60)

  let updateProperty: string[] = []
  let newRow: Row = resizeRecord.value

  if (resizeDirection.value === 'right') {
    let newEndDate = dayjs(selectedDateRange.value.start).add(day, 'day').add(hour, 'hour').add(minutes, 'minute')
    updateProperty = [toCol.title!]

    // If the new end date is before the start date, we set the new end date to the start date
    if (dayjs(newEndDate).isBefore(ogStartDate, 'day')) {
      newEndDate = ogStartDate.clone()
    }

    if (!newEndDate.isValid()) return

    newRow = {
      ...resizeRecord.value,
      row: {
        ...resizeRecord.value.row,
        [toCol.title!]: newEndDate.format('YYYY-MM-DD HH:mm:ssZ'),
      },
    }
  } else if (resizeDirection.value === 'left') {
    let newStartDate = dayjs(selectedDateRange.value.start).add(day, 'day').add(hour, 'hour').add(minutes, 'minute')
    updateProperty = [fromCol.title!]

    // If the new start date is after the end date, we set the new start date to the end date
    if (dayjs(newStartDate).isAfter(ogEndDate)) {
      newStartDate = dayjs(dayjs(ogEndDate)).clone()
    }
    if (!newStartDate) return

    newRow = {
      ...resizeRecord.value,
      row: {
        ...resizeRecord.value.row,
        [fromCol.title!]: dayjs(newStartDate).format('YYYY-MM-DD HH:mm:ssZ'),
      },
    }
  }

  const newPk = extractPkFromRow(newRow.row, meta.value!.columns!)

  formattedData.value = formattedData.value.map((r) => {
    const pk = extractPkFromRow(r.row, meta.value!.columns!)
    return pk === newPk ? newRow : r
  })
  useDebouncedRowUpdate(newRow, updateProperty, false)
}

const onResizeEnd = () => {
  resizeInProgress.value = false
  resizeDirection.value = null
  resizeRecord.value = null
  document.removeEventListener('mousemove', onResize)
  document.removeEventListener('mouseup', onResizeEnd)
}

const onResizeStart = (direction: 'right' | 'left', event: MouseEvent, record: Row) => {
  if (!isUIAllowed('dataEdit')) return
  resizeInProgress.value = true
  resizeDirection.value = direction
  resizeRecord.value = record
  document.addEventListener('mousemove', onResize)
  document.addEventListener('mouseup', onResizeEnd)
}

// We calculate the new row based on the mouse position and update the record
// We also update the sidebar data if the dropped from the sidebar
const calculateNewRow = (
  event: MouseEvent,
  updateSideBar?: boolean,
  skipChangeCheck?: boolean,
): {
  newRow: Row | null
  updatedProperty: string[]
  skipChangeCheck?: boolean
} => {
  const { width, left, top } = container.value.getBoundingClientRect()

  const { scrollHeight } = container.value

  const percentX = (event.clientX - left - window.scrollX) / width
  const percentY = (event.clientY - top + container.value.scrollTop - 36.8) / scrollHeight

  const fromCol = dragRecord.value.rowMeta.range?.fk_from_col
  const toCol = dragRecord.value.rowMeta.range?.fk_to_col

  if (!fromCol) return { newRow: null, updatedProperty: [] }

  const day = Math.max(0, Math.min(6, Math.floor(percentX * 7)))
  const hour = Math.max(0, Math.min(23, Math.floor(percentY * 24)))

  const minutes = Math.round(((percentY * 24 * 60) % 60) / 15) * 15

  const newStartDate = dayjs(selectedDateRange.value.start).add(day, 'day').add(hour, 'hour').add(minutes, 'minute')
  if (!newStartDate) return { newRow: null, updatedProperty: [] }

  let endDate
  const updatedProperty = [fromCol.title!]

  const newRow = {
    ...dragRecord.value,
    row: {
      ...dragRecord.value.row,
      [fromCol.title!]: dayjs(newStartDate).utc().format('YYYY-MM-DD HH:mm:ssZ'),
    },
  }

  if (toCol) {
    const fromDate = dragRecord.value.row[fromCol.title!] ? dayjs(dragRecord.value.row[fromCol.title!]) : null
    const toDate = dragRecord.value.row[toCol.title!] ? dayjs(dragRecord.value.row[toCol.title!]) : null

    if (fromDate && toDate) {
      endDate = dayjs(newStartDate).add(toDate.diff(fromDate, 'day'), 'day')
    } else if (fromDate && !toDate) {
      endDate = dayjs(newStartDate).endOf('day')
    } else if (!fromDate && toDate) {
      endDate = dayjs(newStartDate).endOf('day')
    } else {
      endDate = newStartDate.clone()
    }

    newRow.row[toCol.title!] = dayjs(endDate).utc().format('YYYY-MM-DD HH:mm:ssZ')
    updatedProperty.push(toCol.title!)
  }

  // If from and to columns of the dragRecord and the newRow are the same, we don't manipulate the formattedRecords and formattedSideBarData. This removes unwanted computation
  if (dragRecord.value.row[fromCol.title!] === newRow.row[fromCol.title!] && !skipChangeCheck) {
    return { newRow: null, updatedProperty: [] }
  }

  if (!newRow) return { newRow: null, updatedProperty: [] }

  const newPk = extractPkFromRow(newRow.row, meta.value!.columns!)

  if (updateSideBar) {
    formattedData.value = [...formattedData.value, newRow]
    formattedSideBarData.value = formattedSideBarData.value.filter((r) => {
      const pk = extractPkFromRow(r.row, meta.value!.columns!)
      return pk !== newPk
    })
  } else {
    formattedData.value = formattedData.value.map((r) => {
      const pk = extractPkFromRow(r.row, meta.value!.columns!)
      return pk === newPk ? newRow : r
    })
    dragRecord.value = {
      ...dragRecord.value,
      row: newRow.row,
    }
  }

  return { newRow, updatedProperty }
}

const onDrag = (event: MouseEvent) => {
  if (!isUIAllowed('dataEdit') || !scrollContainer.value || !dragRecord.value) return

  const containerRect = scrollContainer.value.getBoundingClientRect()
  const scrollBottomThreshold = 20

  if (event.clientY > containerRect.bottom - scrollBottomThreshold) {
    scrollContainer.value.scrollTop += 20
  } else if (event.clientY < containerRect.top + scrollBottomThreshold) {
    scrollContainer.value.scrollTop -= 20
  }

  calculateNewRow(event)
}

const stopDrag = (event: MouseEvent) => {
  if (!isUIAllowed('dataEdit') || !isDragging.value || !container.value || !dragRecord.value) return

  event.preventDefault()
  clearTimeout(dragTimeout.value!)

  const { newRow, updatedProperty } = calculateNewRow(event, false, true)

  // We set the visibility and opacity of the records back to normal
  const allRecords = document.querySelectorAll('.draggable-record')
  allRecords.forEach((el) => {
    el.style.visibility = ''
    el.style.opacity = '100%'
  })

  if (newRow) {
    updateRowProperty(newRow, updatedProperty, false)
  }
  $e('c:calendar:week:drag-record')

  document.removeEventListener('mousemove', onDrag)
  document.removeEventListener('mouseup', stopDrag)
}

const dragStart = (event: MouseEvent, record: Row) => {
  if (!isUIAllowed('dataEdit')) return
  if (resizeInProgress.value) return
  let target = event.target as HTMLElement

  isDragging.value = false

  dragTimeout.value = setTimeout(() => {
    isDragging.value = true
    while (!target.classList.contains('draggable-record')) {
      target = target.parentElement as HTMLElement
    }

    const allRecords = document.querySelectorAll('.draggable-record')
    allRecords.forEach((el) => {
      if (!el.getAttribute('data-unique-id').includes(record.rowMeta.id!)) {
        // el.style.visibility = 'hidden'
        el.style.opacity = '30%'
      }
    })

    isDragging.value = true
    dragRecord.value = record

    document.addEventListener('mousemove', onDrag)
    document.addEventListener('mouseup', stopDrag)
  }, 200)

  const onMouseUp = () => {
    clearTimeout(dragTimeout.value!)
    document.removeEventListener('mouseup', onMouseUp)
    if (!isDragging.value) {
      emits('expandRecord', record)
    }
  }

  document.addEventListener('mouseup', onMouseUp)
}

const dropEvent = (event: DragEvent) => {
  if (!isUIAllowed('dataEdit') || !container.value) return
  event.preventDefault()

  const data = event.dataTransfer?.getData('text/plain')
  if (data) {
    const {
      record,
    }: {
      record: Row
    } = JSON.parse(data)

    dragRecord.value = record

    const { newRow, updatedProperty } = calculateNewRow(event, true)

    if (newRow) {
      updateRowProperty(newRow, updatedProperty, false)
      $e('c:calendar:day:drag-record')
    }
  }
}

const viewMore = (hour: dayjs.Dayjs) => {
  sideBarFilterOption.value = 'selectedHours'
  selectedTime.value = hour
  showSideMenu.value = true
}

const isOverflowAcrossHourRange = (hour: dayjs.Dayjs) => {
  if (!recordsAcrossAllRange.value || !recordsAcrossAllRange.value.gridTimeMap) return { isOverflow: false, overflowCount: 0 }
  const { gridTimeMap } = recordsAcrossAllRange.value
  const dayIndex = getDayIndex(hour)
  const startMinute = hour.hour() * 60 + hour.minute()
  const endMinute = hour.hour() * 60 + hour.minute() + 59
  let overflowCount = 0

  for (let minute = startMinute; minute <= endMinute; minute++) {
    const recordCount = gridTimeMap.get(dayIndex)?.get(minute)?.count ?? 0
    overflowCount = Math.max(overflowCount, recordCount)
  }

  return { isOverflow: overflowCount - 3 > 0, overflowCount: overflowCount - 3 }
}

// TODO: Add Support for multiple ranges when multiple ranges are supported
const addRecord = (date: dayjs.Dayjs) => {
  if (!isUIAllowed('dataEdit') || !calendarRange.value) return
  const fromCol = calendarRange.value[0].fk_from_col
  if (!fromCol) return
  const newRecord = {
    row: {
      [fromCol.title!]: date.format('YYYY-MM-DD HH:mm:ssZ'),
    },
  }
  emits('newRecord', newRecord)
}

watch(
  () => recordsAcrossAllRange.value,
  () => {
    if (dragRecord.value) return
    const records = document.querySelectorAll('.draggable-record')
    if (records.length) records.item(0)?.scrollIntoView({ behavior: 'smooth', block: 'center' })
    else document.querySelectorAll('.nc-calendar-day-hour').item(9)?.scrollIntoView({ behavior: 'smooth', block: 'center' })
  },
  { immediate: true },
)
</script>

<template>
  <div
    ref="scrollContainer"
    class="h-[calc(100vh-5.4rem)] prevent-select relative flex w-full overflow-y-auto nc-scrollbar-md"
    data-testid="nc-calendar-week-view"
    @drop="dropEvent"
  >
    <div class="flex sticky h-6 z-1 top-0 pl-16 bg-gray-50 w-full">
      <div
        v-for="date in datesHours"
        :key="date[0].toISOString()"
        :class="{
          'text-brand-500': date[0].isSame(dayjs(), 'date'),
        }"
        class="w-1/7 text-center text-[10px] font-semibold leading-4 flex items-center justify-center uppercase text-gray-500 w-full py-1 border-gray-200 last:border-r-0 border-b-1 border-l-1 border-r-0 bg-gray-50"
      >
        {{ dayjs(date[0]).format('DD ddd') }}
      </div>
    </div>
    <div class="absolute bg-white w-16 z-1">
      <div
        v-for="(hour, index) in datesHours[0]"
        :key="index"
        class="h-13 first:mt-0 pt-7.1 nc-calendar-day-hour text-right pr-2 font-semibold text-xs text-gray-400 py-1"
      >
        {{ hour.format('hh a') }}
      </div>
    </div>
    <div ref="container" class="absolute ml-16 flex w-[calc(100%-64px)]">
      <div v-for="(date, index) in datesHours" :key="index" class="h-full w-1/7 mt-7.1" data-testid="nc-calendar-week-day">
        <div
          v-for="(hour, hourIndex) in date"
          :key="hourIndex"
          :class="{
            'border-1 !border-brand-500 bg-gray-50': hour.isSame(selectedTime, 'hour'),
            '!bg-gray-50': hour.get('day') === 0 || hour.get('day') === 6,
          }"
          class="text-center relative transition h-13 text-sm text-gray-500 w-full hover:bg-gray-50 py-1 border-transparent border-1 border-x-gray-100 border-t-gray-100 border-l-gray-200"
          data-testid="nc-calendar-week-hour"
          @dblclick="addRecord(hour)"
          @click="
            () => {
              selectedTime = hour
              dragRecord = undefined
            }
          "
        >
          <NcButton
            v-if="isOverflowAcrossHourRange(hour).isOverflow"
            v-e="`['c:calendar:week-view-more']`"
            class="!absolute bottom-1 text-center w-15 ml-auto inset-x-0 z-3 text-gray-500"
            size="xxsmall"
            type="secondary"
            @click="viewMore(hour)"
          >
            <span class="text-xs">
              +
              {{ isOverflowAcrossHourRange(hour).overflowCount }}
              more
            </span>
          </NcButton>
        </div>
      </div>

      <div
        class="absolute pointer-events-none inset-0 overflow-hidden !mt-[29px]"
        data-testid="nc-calendar-week-record-container"
      >
        <template v-for="(record, rowIndex) in recordsAcrossAllRange.records" :key="rowIndex">
          <div
            v-if="record.rowMeta.style?.display !== 'none'"
            :data-testid="`nc-calendar-week-record-${record.row[displayField!.title!]}`"
            :data-unique-id="record.rowMeta!.id"
            :style="record.rowMeta!.style "
            class="absolute transition draggable-record w-1/7 group cursor-pointer pointer-events-auto"
            @mousedown.stop="dragStart($event, record)"
            @mouseleave="hoverRecord = null"
            @mouseover="hoverRecord = record.rowMeta.id"
            @dragover.prevent
          >
            <LazySmartsheetRow :row="record">
              <LazySmartsheetCalendarVRecordCard
                :hover="hoverRecord === record.rowMeta.id || record.rowMeta.id === dragRecord?.rowMeta?.id"
                :position="record.rowMeta!.position"
                :resize="!!record.rowMeta.range?.fk_to_col && isUIAllowed('dataEdit')"
                :record="record"
                color="blue"
                :selected="record.rowMeta!.id === dragRecord?.rowMeta?.id"
                @resize-start="onResizeStart"
              >
                <template v-for="(field, id) in fields" :key="id">
                  <LazySmartsheetPlainCell
                    v-if="!isRowEmpty(record, field!)"
                    v-model="record.row[field!.title!]"
                    class="text-xs"
                    :bold="getFieldStyle(field).bold"
                    :column="field"
                    :italic="getFieldStyle(field).italic"
                    :underline="getFieldStyle(field).underline"
                  />
                </template>
                <template #time>
                  <div class="text-xs font-medium text-gray-400">
                    {{ dayjs(record.row[record.rowMeta.range?.fk_from_col!.title!]).format('h:mm a') }}
                  </div>
                </template>
              </LazySmartsheetCalendarVRecordCard>
            </LazySmartsheetRow>
          </div>
        </template>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.prevent-select {
  -webkit-user-select: none;
  -ms-user-select: none;
  user-select: none;
}
</style>
