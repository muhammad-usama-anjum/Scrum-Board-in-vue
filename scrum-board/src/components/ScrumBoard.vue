<template>
  <div class="scrum-board">
    <h1>AGILE SCRUM BOARD</h1>
    <div class="md">
      <div class="action-buttons">
        <button @click="showAddTaskForm" class="add-task-btn">Add Task</button>
        <div class="action-buttons-right">
          <button @click="exportTasks" class="export-tasks-btn">Export Tasks</button>
          <button @click="triggerFileInput" class="import-tasks-btn">Import Tasks</button>
          <input
            type="file"
            ref="fileInput"
            @change="importTasks"
            style="display: none"
            accept=".json"
          />
        </div>
      </div>
    </div>

    <div class="columns-container">
      <div
        v-for="(column, index) in columns"
        :key="index"
        class="column"
        @dragover.prevent
        @drop="onDrop(index)"
      >
        <h2>{{ column.title }} ({{ column.tasks.length }})</h2>
        <div v-if="column.title === 'Backlog'" class="search-bar">
          <input v-model="searchQuery" placeholder="Search tasks..." />
        </div>
        <div class="tasks">
          <div
            v-for="(task, taskIndex) in filteredTasks(column.tasks)"
            :key="taskIndex"
            class="task"
            :class="task.priority"
            draggable="true"
            @dragstart="onDragStart(task, index, taskIndex)"
            @dragend="onDragEnd"
            @click="openTaskDetails(task)"
          >
            <h3>{{ task.title }}</h3>
            <p>Assigned to: {{ task.assignee }}</p>
            <p>Priority: {{ task.priority.charAt(0).toUpperCase() + task.priority.slice(1) }}</p>
            <transition name="fade">
              <div v-if="activeTask === task">
                <p>Description: {{ task.description }}</p>
                <p>Due Date: {{ task.dueDate }}</p>
                <p>Status: {{ task.status }}</p>
                <p>Spent Time: {{ task.spentTime }} hours</p>
              </div>
            </transition>
            <button @click.stop="openEditForm(task)" class="edit">Edit Task</button>
          </div>
        </div>
      </div>
    </div>

    <div v-if="showForm" class="task-form-overlay">
      <div class="task-form">
        <h2>Add New Task</h2>
        <form @submit.prevent="submitTask">
          <label for="task-title">Title:</label>
          <input type="text" id="task-title" v-model="task.title" required />

          <label for="task-description">Description:</label>
          <textarea id="task-description" v-model="task.description" required></textarea>

          <label for="task-assignee">Assignee:</label>
          <input type="text" id="task-assignee" v-model="task.assignee" required />

          <label for="task-due-date">Due Date:</label>
          <input type="date" id="task-due-date" v-model="task.dueDate" :min="todayDate" required />

          <label for="task-spent-time">Spent Time (hours):</label>
          <input
            type="number"
            id="task-spent-time"
            v-model.number="task.spentTime"
            :min="0"
            step="0.01"
            required
          />

          <div class="task-priority">
            <label>Priority:</label>
            <div class="priority-options">
              <label>
                <input type="radio" name="priority" value="low" v-model="task.priority" />
                Low
              </label>
              <label>
                <input type="radio" name="priority" value="normal" v-model="task.priority" />
                Normal
              </label>
              <label>
                <input type="radio" name="priority" value="high" v-model="task.priority" />
                High
              </label>
            </div>
          </div>

          <div class="form-buttons">
            <button type="button" @click="closeForm">Cancel</button>
            <button type="submit">Submit</button>
          
          </div>
        </form>
      </div>
    </div>

    <div v-if="editingTask" class="task-form-overlay">
      <div class="task-form">
        <h2>Edit Task</h2>
        <form @submit.prevent="submitEditedTask">
          <label for="edit-task-title">Title:</label>
          <input type="text" id="edit-task-title" v-model="editedTask.title" required />

          <label for="edit-task-description">Description:</label>
          <textarea id="edit-task-description" v-model="editedTask.description" required></textarea>

          <label for="edit-task-assignee">Assignee:</label>
          <input type="text" id="edit-task-assignee" v-model="editedTask.assignee" required />

          <label for="edit-task-due-date">Due Date:</label>
          <input type="date" id="edit-task-due-date" v-model="editedTask.dueDate" required />

          <label for="edit-task-spent-time">Spent Time (hours):</label>
          <input
            type="number"
            id="edit-task-spent-time"
            v-model.number="editedTask.spentTime"
            :min="0"
            step="0.01"
            required
          />

          <div class="edit-task-priority">
  <label>Priority:</label>
  <div class="priority-options">
    <label>
      <input type="radio" name="edit-task-priority" value="low" v-model="editedTask.priority" />
      Low
    </label>
    <label>
      <input type="radio" name="edit-task-priority" value="normal" v-model="editedTask.priority" />
      Normal
    </label>
    <label>
      <input type="radio" name="edit-task-priority" value="high" v-model="editedTask.priority" />
      High
    </label>
  </div>
</div>


          <div class="form-buttons">
            <button type="button" @click="cancelEdit">Cancel</button>
            <button type="submit">Submit</button>
           
          </div>
        </form>
      </div>
    </div>

    <div v-if="showModal" class="modal-overlay">
      <div class="modal-content">
        <h2>{{ modalTask.title }}</h2>
        <p><strong>Description:</strong> {{ modalTask.description }}</p>
        <p><strong>Assignee:</strong> {{ modalTask.assignee }}</p>
        <p><strong>Due Date:</strong> {{ modalTask.dueDate }}</p>
        <p><strong>Status:</strong> {{ modalTask.status }}</p>
        <p><strong>Spent Time:</strong> {{ modalTask.spentTime }} hours</p>
        <p>
          <strong>Priority:</strong>
          {{ modalTask.priority.charAt(0).toUpperCase() + modalTask.priority.slice(1) }}
        </p>
        <button @click="closeModal">Close</button>
      </div>
    </div>
  </div>
</template>

<script>
import { v4 as uuidv4 } from 'uuid'
export default {
  data() {
    return {
      searchQuery: '',
      showForm: false,
      task: {
        id: '', // Add an ID field
        title: '',
        description: '',
        assignee: '',
        dueDate: '',
        status: 'Backlog',
        spentTime: 0,
        priority: 'normal'
      },
      editingTask: null,
      editedTask: {
        id: '', // Add an ID field
        title: '',
        description: '',
        assignee: '',
        dueDate: '',
        status: '',
        spentTime: '',
        priority: 'normal'
      },
      activeTask: null,
      showModal: false,
      modalTask: null,
      columns: [
        { title: 'Backlog', tasks: [] },
        { title: 'Open', tasks: [] },
        { title: 'New Tasks', tasks: [] },
        { title: 'In Progress', tasks: [] },
        { title: 'Feedback Needed', tasks: [] },
        { title: 'Ready for Testing', tasks: [] },
        { title: 'QA in Progress', tasks: [] }
      ],
      dragStartColumnIndex: null,
      dragStartTaskIndex: null,
      todayDate: new Date().toISOString().split('T')[0]
    }
  },
  computed: {
    filteredTasks() {
      return (tasks) => {
        if (!this.searchQuery) {
          return tasks
        }
        const lowerCaseQuery = this.searchQuery.toLowerCase()
        return tasks.filter((task) => task.title.toLowerCase().includes(lowerCaseQuery))
      }
    }
  },
  methods: {
    onDragStart(task, columnIndex, taskIndex) {
      this.dragStartColumnIndex = columnIndex
      this.dragStartTaskIndex = taskIndex
    },
    onDragEnd() {
      this.dragStartColumnIndex = null
      this.dragStartTaskIndex = null
    },
    onDrop(dropColumnIndex) {
      if (this.dragStartColumnIndex !== null && this.dragStartTaskIndex !== null) {
        const draggedTask = this.columns[this.dragStartColumnIndex].tasks.splice(
          this.dragStartTaskIndex,
          1
        )[0]
        draggedTask.status = this.columns[dropColumnIndex].title
        this.columns[dropColumnIndex].tasks.push(draggedTask)
        this.saveTasksToLocalStorage()
      }
    },
    showAddTaskForm() {
      this.showForm = true
    },
    closeForm() {
      this.showForm = false
    },
    submitTask() {
      if (!this.task.dueDate || new Date(this.task.dueDate) < new Date(this.todayDate)) {
        alert('The due date cannot be in the past.')
        return
      }

      if (this.task.spentTime < 0) {
        alert('Spent time cannot be less than 0.')
        return
      }

      this.task.id = uuidv4() // Generate a unique ID for the task
      this.columns[0].tasks.unshift({ ...this.task })
      this.resetTaskForm()
      this.showForm = false
      this.saveTasksToLocalStorage()
    },

    openTaskDetails(task) {
      this.modalTask = task
      this.showModal = true
    },
    openEditForm(task) {
      this.editedTask = { ...task }
      this.editingTask = task
    },
    submitEditedTask() {
      Object.assign(this.editingTask, this.editedTask)
      this.editingTask = null
      this.editedTask = this.resetEditedTaskForm()
      this.saveTasksToLocalStorage()
    },
    cancelEdit() {
      this.editingTask = null
      this.resetEditedTaskForm()
    },
    resetTaskForm() {
      this.task = {
        id: '', // Reset ID
        title: '',
        description: '',
        assignee: '',
        dueDate: '',
        status: 'Backlog',
        spentTime: 0,
        priority: 'normal'
      }
    },
    resetEditedTaskForm() {
      return {
        id: '', // Reset ID
        title: '',
        description: '',
        assignee: '',
        dueDate: '',
        status: 'Backlog',
        spentTime: 0,
        priority: 'normal'
      }
    },
    closeModal() {
      this.showModal = false
      this.modalTask = null
    },
    exportTasks() {
      const data = JSON.stringify(this.columns)
      const blob = new Blob([data], { type: 'application/json' })
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = 'tasks.json'
      a.click()
      URL.revokeObjectURL(url)
    },
    triggerFileInput() {
      this.$refs.fileInput.click()
    },
    importTasks(event) {
      const file = event.target.files[0]
      if (!file) return

      const reader = new FileReader()
      reader.onload = (e) => {
        try {
          const importedData = JSON.parse(e.target.result)

          if (Array.isArray(importedData)) {
            // Create a map of existing tasks by ID for quick lookup
            const existingTasksMap = new Map()
            this.columns.forEach((column) => {
              column.tasks.forEach((task) => {
                existingTasksMap.set(task.id, task)
              })
            })

            // Add new tasks or update existing tasks
            this.columns.forEach((column, index) => {
              if (importedData[index]) {
                importedData[index].tasks.forEach((importedTask) => {
                  const existingTask = existingTasksMap.get(importedTask.id)

                  if (!existingTask) {
                    // If the task does not exist, add it
                    column.tasks.push(importedTask)
                  } else if (JSON.stringify(importedTask) !== JSON.stringify(existingTask)) {
                    // If the task exists but has changed, update it
                    Object.assign(existingTask, importedTask)
                  }
                })
              }
            })

            this.saveTasksToLocalStorage()
          } else {
            alert('Invalid data format. Make sure the JSON file contains an array of columns.')
          }
        } catch (error) {
          alert('Invalid JSON file.')
        }
      }
      reader.readAsText(file)
    },

    saveTasksToLocalStorage() {
      localStorage.setItem('scrumBoardColumns', JSON.stringify(this.columns))
    },
    loadTasksFromLocalStorage() {
      const data = localStorage.getItem('scrumBoardColumns')
      if (data) {
        try {
          this.columns = JSON.parse(data)
        } catch (error) {
          console.error('Error parsing localStorage data:', error)
        }
      }
    }
  },
  created() {
    this.loadTasksFromLocalStorage()
  }
}
</script>

  <style scoped>
.scrum-board {
  display: flex;
  flex-direction: column;
  align-items: center;
  background-color: black;
  height: 100vh;
  justify-content: flex-start;
  padding-top: 20px;
  box-sizing: border-box;
}

.scrum-board h1 {
  margin-top: 20px;
  color: white;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  font-size: 1.5rem;
  margin-bottom: 5px;
  font-weight: bold;
}

.action-buttons {
  width: 100%;
  display: flex;
  justify-content: space-between;
  margin-bottom: 20px;
  padding: 0 20px;
}

.action-buttons-right {
  display: flex;
  align-items: center;
}

.md {
  height: 50px;
  width: 100%;
  background-color: #292b2c;
  margin: 20px;
  padding-top: 6px;
}

@keyframes pulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.05);
  }
  100% {
    transform: scale(1);
  }
}

.edit {
  padding: 5px 10px;
  font-size: 12px;
  background-color: #000103;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-right: 5px;
  margin-left: 10px;
  font-weight: bold;
}

.edit:hover {
  animation: pulse 0.5s ease-in-out forwards;
  background-color: #343435;
}

.add-task-btn {
  justify-content: flex-start;
  padding: 10px 20px;
  font-size: 16px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-right: 20px;
  margin-left: 10px;
  font-weight: bold;
}

.add-task-btn:hover {
  animation: pulse 0.5s ease-in-out forwards;
  background-color: #195b86;
}

.import-tasks-btn {
  padding: 10px 20px;
  font-size: 16px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-right: 10px;
  font-weight: bold;
  float: right;
}

.import-tasks-btn:hover {
  animation: pulse 0.5s ease-in-out forwards;
  background-color: #195b86;
}

.export-tasks-btn {
  padding: 10px 20px;
  font-size: 16px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-right: 20px;
  font-weight: bold;
}

.export-tasks-btn:hover {
  animation: pulse 0.5s ease-in-out forwards;
  background-color: #195b86;
}

.columns-container {
  display: flex;
  justify-content: space-around;
  width: 100%;
  margin-bottom: 20px;
  margin: 0%;
  padding: 0%;
}

.column {
  flex: 1;
  min-width: 187px;
  max-width: 200px;
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 10px;
  background-color: #f9f9f9;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  box-sizing: border-box;
}

.column h2 {
  text-align: center;
  font-size: 16px;
  margin-bottom: 10px;
  border-bottom: 1px solid #ccc;
  padding-bottom: 5px;
  font-weight: bold;
}

.search-bar {
  margin-bottom: 10px;
}

.search-bar input {
  width: 100%;
  padding: 6px 10px 6px 20px;
  border: 1px solid #ccc;
  border-radius: 4px;
  background-image: url('https://img.icons8.com/?size=100&id=84039&format=png&color=000000');
  background-repeat: no-repeat;
  background-size: 12%;
  background-position: 2px 4px;
}

.tasks {
  margin-top: 10px;
}

.task {
  word-wrap: break-word;
  background-color: #f5f5f5;
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 10px;
  margin-bottom: 10px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  cursor: pointer;
  transition: background-color 0.3s;
  
}

.edit-task-priority {
  margin: 10px 0;
}

.edit-task-priority label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

.priority-options {
  display: flex;
  gap: 10px;
}

.priority-options label {
  display: flex;
  align-items: center;
  cursor: pointer;
}

.priority-options input[type="radio"] {
  margin-right: 5px;
  cursor: pointer;
}

/* Optional: Style for selected radio button */
.priority-options input[type="radio"]:checked {
  accent-color: #007bff;
}


.task.low {
  background-color: #e1e452;
}

.task.low:hover {
  animation: pulse 0.5s ease-in-out forwards;
  background-color: #fff020;
}

.task.high {
  background-color: #f59292;
}

.task.high:hover {
  animation: pulse 0.5s ease-in-out forwards;
  background-color: #f53a3a;
}

.task.normal {
  background-color: #cce5ff;
}

.task.normal:hover {
  animation: pulse 0.5s ease-in-out forwards;
  background-color: #47b6f7;
}

.task h3 {
  font-size: 14px;
  margin: 0 0 5px;
  font-weight: bold;
  word-wrap: break-word;
  overflow-wrap: break-word;
  white-space: nowrap; /* Ensure text wraps within the title */
  overflow: hidden;
  text-overflow: ellipsis;
}

.task p {
  margin: 0;
  font-size: 12px;
}

.task-details {
  cursor: pointer;
}

.task-details:hover {
  background-color: #e9ecef;
}

.task-form-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.task-form {
  width: 50%;
  background-color: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  max-height: 80vh; /* Adjust this value as needed */
  overflow-y: auto;
}

.task-form h2 {
  text-align: center;
  font-size: 18px;
  margin-bottom: 10px;
}

form {
  display: flex;
  flex-direction: column;
  align-items: center;
}

label {
  margin-top: 10px;
  font-weight: bold;
}

input,
textarea,
select {
  width: 100%;
  padding: 8px;
  margin-top: 5px;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-sizing: border-box;
}

.form-buttons {
  margin-top: 10px;
  display: flex;
  justify-content: space-around;
  width: 100%;
}

.form-buttons button {
  padding: 8px 20px;
  font-size: 14px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.form-buttons button[type='submit'] {
  background-color: #007bff;
  color: white;
}

.form-buttons button[type='button'] {
  background-color: #6c757d;
  color: white;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s;
}

.fade-enter,
.fade-leave-to {
  opacity: 0;
}

.task-priority {
  margin: 10px 0;
}

.task-priority label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

.priority-options {
  display: flex;
  gap: 10px;
}

.priority-options label {
  display: flex;
  align-items: center;
  cursor: pointer;
}

.priority-options input[type="radio"] {
  margin-right: 5px;
  cursor: pointer;
}

/* Optional: Style for selected radio button */
.priority-options input[type="radio"]:checked {
  accent-color: #007bff;
}


.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000; 
}

.modal-content {
  width: 80%;
  max-width: 600px; 
  background-color: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.3);
  max-height: 80vh;
  overflow-y: auto; 
  word-wrap: break-word;
}

.modal-content h2 {
  text-align: center;
  font-size: 20px;
  margin-bottom: 10px;
}

.modal-content p {
  margin-bottom: 5px;
}

.modal-content button {
  padding: 8px 20px;
  font-size: 14px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  background-color: #007bff;
  color: white;
}

.modal-content button:hover {
  background-color: #0056b3;
}

@media (max-width: 768px) {
  .scrum-board h1 {
    font-size: 3rem;
  }

  .add-task-btn,
  .import-tasks-btn,
  .export-tasks-btn,
  .edit {
    padding: 5px 10px;
    font-size: 12px;
  }

  .task-form {
    width: 90%;
  }

  .form-buttons {
    flex-direction: column;
    align-items: center;
  }

  .form-buttons button {
    width: 100%;
    margin-bottom: 5px;
  }
}

@media (max-width: 576px) {
  .scrum-board h1 {
    font-size: 2.5rem;
  }

  .columns-container {
    flex-direction: column;
    align-items: center;
  }

  .column {
    width: 100%;
    max-width: 100%;
    margin: 5px 0;
  }

  .task-form {
    width: 95%;
  }
}
</style>
  