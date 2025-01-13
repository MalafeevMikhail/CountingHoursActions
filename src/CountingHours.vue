<template>
  <div class="container">
    <!-- Tags Management -->
    <h1 class="project-title">Добро пожаловать!</h1>
    <p class="project-description">
      Ниже представлены теги, по которым вы можете считать свои часы<br />
      Так же вы можете добавить <b>свои теги</b> с <b>цветами</b> ^-^
    </p>
    <div class="tags-section">
      <h2>Ваши теги</h2>
      <div class="tag-input">
        <input
          v-model="newTag.name"
          type="text"
          placeholder="Введите имя тега"
        />
        <input v-model="newTag.color" type="color" />
        <button @click="addTag">Добавить тег</button>
      </div>
      <div>
        <span class="error" v-if="isNotEnterName"
          >Необходимо указать имя тега</span
        >
        <span class="error" v-if="isHaveThisNameTag"
          >Данное имя тега уже добавлено</span
        >
        <span class="error" v-if="isHaveThisColor"
          >Данный цвет уже используется</span
        >
      </div>
      <div class="tags-list">
        <span
          v-for="(tag, idx) in tags"
          :key="tag.name"
          class="tag"
          :style="{ backgroundColor: tag.color }"
        >
          {{ tag.name }}
          <span>{{ formatDuration(getTagDuration(tag.name)) }}</span>
          <div class="close" @click="deleteTag(idx)"></div>
        </span>
      </div>
      <span class="error" v-if="isCanNotDeleteThisTagColor"
        >Нельзя удалить данный тег, так как он используется в планировании</span
      >
    </div>

    <!-- Calendar Grid -->
    <div class="calendar">
      <div class="time-column">
        <div v-for="hour in 24" :key="hour - 1" class="hour-slot">
          {{ formatHour(hour - 1) }}
        </div>
      </div>

      <div
        class="events-grid"
        @mousedown="startSelection"
        @mousemove="updateSelection"
        @mouseup="endSelection"
      >
        <div v-for="hour in 24" :key="hour - 1" class="grid-hour"></div>

        <!-- Events -->
        <div
          v-for="event in events"
          :key="event.id"
          :style="getEventStyle(event)"
          class="event"
          @click="editEvent(event)"
        >
          <div class="event-title">{{ event.title }}</div>
          <div class="event-time">{{ formatEventTime(event) }}</div>
          <div class="event-tag">{{ event.tag }}</div>
        </div>

        <!-- Selection overlay -->
        <div v-if="selecting" :style="selectionStyle" class="selection"></div>
      </div>
    </div>

    <!-- Modal -->
    <div v-if="showModal" class="modal-overlay">
      <div class="modal">
        <h3>
          {{
            editingEvent ? "Редактирование мероприяти" : "Создание мероприятия"
          }}
        </h3>

        <input
          v-model="newEvent.title"
          type="text"
          placeholder="Описание мероприятия"
        />

        <div class="time-inputs">
          <div>
            <label>Время начала</label>
            <input
              v-model="newEvent.startTime"
              type="time"
              @input="validateTimes"
            />
          </div>
          <div>
            <label>Время окончания</label>
            <input
              v-model="newEvent.endTime"
              type="time"
              @input="validateTimes"
            />
          </div>
        </div>

        <select v-model="newEvent.tag">
          <option value="">Выберите тег</option>
          <option v-for="tag in tags" :key="tag.name" :value="tag.name">
            {{ tag.name }}
          </option>
        </select>

        <div class="modal-buttons">
          <button v-if="editingEvent" @click="deleteEvent" class="delete-btn">
            Удалить
          </button>
          <div>
            <button @click="closeModal" class="cancel-btn">Закрыть</button>
            <button
              @click="saveEvent"
              :disabled="!isValidEvent"
              class="save-btn"
            >
              {{ editingEvent ? "Сохранить" : "Создать" }}
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from "vue";

const tags = ref([
  { name: "Работа", color: "#48BB78" },
  { name: "Учёба", color: "#4A5568" },
]);
const newTag = ref({ name: "", color: "#3182CE" });
const events = ref([]);
const selecting = ref(false);
const selectionStart = ref(null);
const selectionEnd = ref(null);
const showModal = ref(false);
const editingEvent = ref(null);

const defaultEventState = {
  title: "",
  tag: "",
  startTime: "",
  endTime: "",
  start: null,
  end: null,
};

const newEvent = ref({ ...defaultEventState });
const isNotEnterName = ref(false);
const isHaveThisNameTag = ref(false);
const isHaveThisColor = ref(false);
const isCanNotDeleteThisTagColor = ref(false);

// Local Storage
const STORAGE_KEY = "time-tracker";

const saveToLocalStorage = () => {
  localStorage.setItem(
    STORAGE_KEY,
    JSON.stringify({
      tags: tags.value,
      events: events.value,
    })
  );
};

const loadFromLocalStorage = () => {
  const data = localStorage.getItem(STORAGE_KEY);
  if (data) {
    const parsed = JSON.parse(data);
    tags.value = parsed.tags;
    events.value = parsed.events;
  }
};

watch([tags, events], saveToLocalStorage, { deep: true });
onMounted(loadFromLocalStorage);

const addTag = () => {
  if (!newTag.value.name) {
    isNotEnterName.value = true;
    return;
  }
  isNotEnterName.value = false;

  if (tags.value.some((tag) => tag.color === newTag.value.color)) {
    isHaveThisColor.value = true;
    return;
  }
  isHaveThisColor.value = false;

  if (tags.value.some((tag) => tag.name === newTag.value.name)) {
    isHaveThisNameTag.value = true;
    return;
  }
  isHaveThisNameTag.value = false;

  tags.value.push({ ...newTag.value });
  newTag.value = { name: "", color: "#3182CE" };
};

const deleteTag = (idx) => {
  if (idx < 0 || idx > tags.value.length) {
    return;
  }

  let tagObject = tags.value[idx];
  if (events.value.some((item) => item.tag === tagObject.name)) {
    isCanNotDeleteThisTagColor.value = true;
    return;
  }

  isCanNotDeleteThisTagColor.value = false;

  tags.value.splice(idx, 1);
};

const startSelection = (e) => {
  selecting.value = true;
  selectionStart.value = getTimeFromY(e.clientY);
  selectionEnd.value = selectionStart.value;
};

const updateSelection = (e) => {
  if (selecting.value) {
    selectionEnd.value = getTimeFromY(e.clientY);
  }
};

const endSelection = () => {
  if (selecting.value) {
    selecting.value = false;
    const startMinutes = Math.min(selectionStart.value, selectionEnd.value);
    const endMinutes = Math.max(selectionStart.value, selectionEnd.value);

    newEvent.value = {
      ...defaultEventState,
      startTime: minutesToTime(startMinutes),
      endTime: minutesToTime(endMinutes),
      start: startMinutes,
      end: endMinutes,
    };

    showModal.value = true;
  }
};

const minutesToTime = (minutes) => {
  const hours = Math.floor(minutes / 60);
  const mins = minutes % 60;
  return `${hours.toString().padStart(2, "0")}:${mins
    .toString()
    .padStart(2, "0")}`;
};

const timeToMinutes = (time) => {
  const [hours, minutes] = time.split(":").map(Number);
  return hours * 60 + minutes;
};

const validateTimes = () => {
  if (newEvent.value.startTime && newEvent.value.endTime) {
    const start = timeToMinutes(newEvent.value.startTime);
    const end = timeToMinutes(newEvent.value.endTime);

    if (start < end) {
      newEvent.value.start = start;
      newEvent.value.end = end;
    } else {
      newEvent.value.endTime = newEvent.value.startTime;
      newEvent.value.end = start;
    }
  }
};

const saveEvent = () => {
  if (isValidEvent.value) {
    if (editingEvent.value) {
      const index = events.value.findIndex(
        (e) => e.id === editingEvent.value.id
      );
      if (index !== -1) {
        events.value[index] = {
          id: editingEvent.value.id,
          ...newEvent.value,
        };
      }
    } else {
      events.value.push({
        id: Date.now(),
        ...newEvent.value,
      });
    }
    closeModal();
  }
};

const editEvent = (event) => {
  editingEvent.value = event;
  newEvent.value = { ...event };
  showModal.value = true;
};

const deleteEvent = () => {
  if (editingEvent.value) {
    events.value = events.value.filter((e) => e.id !== editingEvent.value.id);
    closeModal();
  }
};

const closeModal = () => {
  showModal.value = false;
  editingEvent.value = null;
  newEvent.value = { ...defaultEventState };
};

const getTimeFromY = (clientY) => {
  const element = event.currentTarget;
  const rect = element.getBoundingClientRect();
  const y = clientY - rect.top;
  const totalMinutes = (y / element.offsetHeight) * 24 * 60;
  return Math.floor(totalMinutes / 15) * 15;
};

const isValidEvent = computed(() => {
  return (
    newEvent.value.title &&
    newEvent.value.tag &&
    newEvent.value.startTime &&
    newEvent.value.endTime &&
    newEvent.value.start < newEvent.value.end
  );
});

const selectionStyle = computed(() => {
  if (!selecting.value) return {};
  const top =
    (Math.min(selectionStart.value, selectionEnd.value) / (24 * 60)) * 100;
  const height =
    (Math.abs(selectionEnd.value - selectionStart.value) / (24 * 60)) * 100;
  return {
    top: `${top}%`,
    height: `${height}%`,
  };
});

const getEventStyle = (event) => {
  const top = (event.start / (24 * 60)) * 100;
  const height = ((event.end - event.start) / (24 * 60)) * 100;
  const tagColor =
    tags.value.find((t) => t.name === event.tag)?.color || "#3182CE";
  return {
    top: `${top}%`,
    height: `${height}%`,
    backgroundColor: tagColor,
  };
};

const getTagDuration = (tagName) => {
  return events.value
    .filter((event) => event.tag === tagName)
    .reduce((total, event) => total + (event.end - event.start), 0);
};

const formatHour = (hour) => {
  return `${hour.toString().padStart(2, "0")}:00`;
};

const formatDuration = (minutes) => {
  const hours = Math.floor(minutes / 60);
  const mins = minutes % 60;
  return `${hours}h ${mins}m`;
};

const formatEventTime = (event) => {
  return `${minutesToTime(event.start)} - ${minutesToTime(event.end)}`;
};
</script>

<style scoped>
.container {
  min-height: 100vh;
  background: #1a1a1a;
  color: white;
  padding: 1rem;
}

.project-title {
  margin-bottom: 20px;
}

.project-description {
  margin-bottom: 20px;
  border-left: 2px solid white;
  padding-left: 10px;
  max-width: 500px;
  line-height: 20px;
}
.tags-section {
  margin-bottom: 1.5rem;
}

.tags-section h2 {
  font-size: 1.5rem;
  font-weight: bold;
  margin-bottom: 0.75rem;
}

.tag-input {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 0.75rem;
}

.input-color-block {
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 5px;
}

.error {
  color: red;
  font-weight: bold;
}

.close {
  width: 10px;
  height: 10px;
  cursor: pointer;
  transition: transform 1s linear;

  -webkit-clip-path: polygon(
    20% 0%,
    0% 20%,
    30% 50%,
    0% 80%,
    20% 100%,
    50% 70%,
    80% 100%,
    100% 80%,
    70% 50%,
    100% 20%,
    80% 0%,
    50% 30%
  );
  clip-path: polygon(
    20% 0%,
    0% 20%,
    30% 50%,
    0% 80%,
    20% 100%,
    50% 70%,
    80% 100%,
    100% 80%,
    70% 50%,
    100% 20%,
    80% 0%,
    50% 30%
  );
  background: black;
}

.close:hover {
  transform: scale(1.1);
}

.tag-input input[type="text"] {
  background: #2d2d2d;
  padding: 0.5rem 0.75rem;
  border-radius: 0.25rem;
  border: none;
  color: white;
  height: 60px;
  font-size: 20px;
}

.tag-input input[type="color"] {
  width: 60px;
  height: 60px;
  background-color: transparent;
  border: none;
  cursor: pointer;
  transition: transform 0.2s linear;
}

.tag-input input[type="color"]:hover {
  transform: scale(1.1);
}

.tag-input button {
  background: #3182ce;
  font-size: 25px;
  font-weight: bold;
  padding: 0.5rem 1rem;
  border-radius: 0.25rem;
  border: none;
  color: white;
  cursor: pointer;
}

.tag-input button:hover {
  background: #2c5282;
}

.tags-list {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 10px;
  margin-bottom: 10px;
}

.tag {
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.875rem;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.calendar {
  position: relative;
  border-top: 1px solid #333;
  display: flex;
}

.time-column {
  position: sticky;
  left: 0;
  width: 4rem;
  background: #1a1a1a;
  z-index: 1;
}

.hour-slot {
  height: 5rem;
  border-bottom: 1px solid #333;
  font-size: 0.875rem;
  color: #666;
  padding-right: 0.5rem;
  text-align: right;
}

.events-grid {
  flex: 1;
  position: relative;
  min-height: calc(24 * 5rem);
}

.grid-hour {
  height: 5rem;
  border-bottom: 1px solid #333;
}

.event {
  position: absolute;
  left: 0;
  right: 0;
  padding: 0.5rem;
  overflow: hidden;
  cursor: pointer;
  color: white;
  transition: opacity 0.2s;
}

.event:hover {
  opacity: 0.9;
}

.event-title {
  font-weight: bold;
  font-size: 0.875rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.event-time {
  font-size: 0.75rem;
}

.event-tag {
  font-size: 0.75rem;
}

.selection {
  position: absolute;
  left: 0;
  right: 0;
  background: rgba(49, 130, 206, 0.5);
}

.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal {
  background: #2d2d2d;
  padding: 1.5rem;
  border-radius: 0.5rem;
  width: 24rem;
}

.modal h3 {
  font-size: 1.25rem;
  font-weight: bold;
  margin-bottom: 1rem;
}

.modal input[type="text"],
.modal input[type="time"],
.modal select {
  width: 100%;
  background: #1a1a1a;
  padding: 0.5rem 0.75rem;
  border-radius: 0.25rem;
  border: 1px solid #333;
  color: white;
  margin-bottom: 0.75rem;
}

.time-inputs {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.75rem;
  margin-bottom: 0.75rem;
}

.time-inputs label {
  display: block;
  font-size: 0.875rem;
  margin-bottom: 0.25rem;
}

.modal-buttons {
  display: flex;
  justify-content: space-between;
  margin-top: 1rem;
}

.modal-buttons button {
  padding: 0.5rem 1rem;
  border-radius: 0.25rem;
  border: none;
  cursor: pointer;
}

.delete-btn {
  background: #e53e3e;
  color: white;
}

.delete-btn:hover {
  background: #c53030;
}

.cancel-btn {
  background: #4a5568;
  color: white;
  margin-right: 0.5rem;
}

.cancel-btn:hover {
  background: #2d3748;
}

.save-btn {
  background: #3182ce;
  color: white;
}

.save-btn:hover {
  background: #2c5282;
}

.save-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
