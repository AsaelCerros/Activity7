<template>
  <section class="todoapp">
    <header class="header">
      <h1>Tareas</h1>
      {{database}}
      <button @click="getDataBase">llamar a la lbd</button>
      <input
          class="new-todo"
          autofocus
          autocomplete="off"
          placeholder="Creaar una nueva tareas"
          v-model="newTodo"
          @keyup.enter="addTodo"
      />
    </header>
    <section class="main" v-show="todos.length">
      <input
          id="toggle-all"
          class="toggle-all"
          type="checkbox"
          v-model="allDone"
      />
      <label for="toggle-all">Marcar todas comocompletas </label>
      <ul class="todo-list">
        <li
            class="todo"
            v-for="todo in filteredTodos"
            :key="todo.id"
            :class="{ completed: todo.completed, editing: todo == editedTodo }"
        >
          <div class="view">
            <input
                class="toggle"
                type="checkbox"
                @click="updateTodo(todo)"
                :checked="todo.completed"
            />
            <label @dblclick="editTodo(todo)">{{ todo.title }}</label>
            <button class="destroy" @click="removeTodo(todo)"></button>
          </div>
          <input
              class="edit"
              type="text"
              v-model="todo.title"
              @blur="doneEdit(todo)"
              @keyup.enter="doneEdit(todo)"
              @keyup.esc="cancelEdit(todo)"
          />
        </li>
      </ul>
    </section>
    <footer class="footer" v-show="todos.length">
      <span class="todo-count">
        <strong v-text="remaining"></strong>
        {{ pluralize("tarea", remaining) }}
      </span>
      <ul class="filters">
        <li>
          <button
              @click="visibility = 'all'"
              :class="{ selected: visibility == 'all' }"
              class="btn"
          >
            Todas
          </button>
        </li>
        <li>
          <button
              @click="visibility = 'active'"
              :class="{ selected: visibility == 'active' }"
              class="btn"
          >
            Activas
          </button>
        </li>
        <li>
          <button
              @click="visibility = 'completed'"
              :class="{ selected: visibility == 'completed' }"
              class="btn"
          >
            Competas
          </button>
        </li>
      </ul>
      <button
          class="clear-completed"
          @click="removeCompleted"
          v-show="todos.length > remaining"
      >
        Limpiar completadas
      </button>
    </footer>
  </section>
  <footer class="info">
    <p>Doble click para editar la tarea</p>
  </footer>
</template>

<script>
export default {
  name: "App",
  data: () => ({
    todos: [],
    newTodo: "",
    editedTodo: null,
    visibility: "all",
    database: null,
  }),

  computed: {
    activeTasks() {
      return this.todos.filter((todo) => !todo.completed);
    },
    filteredTodos() {
      if (this.visibility === "all") {
        return this.todos;
      } else if (this.visibility === "active") {
        return this.activeTasks;
      } else {
        return this.todos.filter((todo) => todo.completed);
      }
    },
    remaining() {
      return this.activeTasks.length;
    },
    allDone: {
      get() {
        return this.remaining === 0;
      },
      set(value) {
        this.todos.forEach((todo) => {
          todo.completed = value;
        });
      },
    },
  },
  async created(){
    this.todos = await this.getTodoStore()
  },
  methods: {
    addTodo() {
      const value = this.newTodo && this.newTodo.trim();
      const todoItem = {
        id: this.todos.length + 1,
        title: value,
        completed: false,
      };

      if (!value) {
        return;
      }
      this.todos.push(todoItem); //SE GUARDA EN EL ARREGLO
      this.saveTodo(todoItem); //SE GUARDA EN INDEXDB
      this.newTodo = "";
    },

    cancelEdit(todo) {
      this.editedTodo = null;
      todo.title = this.beforeEditCache;
    },

    doneEdit(todo) {
      if (!this.editedTodo) {
        return
      }
      this.editedTodo = null
      todo.title = todo.title.trim()
      this.saveTodo({
        ...todo,
        title: todo.title
      })
      if (!todo.title) {
        this.removeTodo(todo)
      }
    },
    editTodo(todo) {
      this.beforeEditCache = todo.title;
      this.editedTodo = todo;
    },

    pluralize(word, count) {
      return word + (count === 1 ? "" : "s");
    },

    removeCompleted() {
      this.todos = this.todos.filter((item) => {
        if (item.completed) {
          this.deleteTodo(item)
        } else {
          return !item.completed
        }
      });
    },

    removeTodo(todo) {
      const index = this.todos.indexOf(todo);
      this.todos.splice(index, 1);
      this.deleteTodo(todo)
    },

    updateTodo(todo) {
      this.todos.find(item => item === todo).completed = !todo.completed
      this.saveTodo({
        ...todo
      })
    },
    async getDataBase(){

      return new Promise((resolve, reject) => {

        if(this.database){
          resolve(this.database)
        }

        let request = window.indexedDB.open('tododb',1);

        request.onerror = event  => {
          this.database = event.target.result
          reject("Error")
        }

        request.onsuccess = event  => {
          this.database = event.target.result
          resolve(this.database)
        }

        request.onupgradeneeded = event  => {
          let database =  event.target.result
          database.createObjectStore('todos', {
            autoIncrement: true,
            keyPath: 'id'
          })

        }
      })


    },
    async getTodoStore() {
      this.database = await this.getDataBase()

      return new Promise((resolve, reject) => {
        const transaction = this.database.transaction('todos','readwrite')
        const store = transaction.objectStore('todos')
        let todoList = []
        store.openCursor().onsuccess = event => {
          const cursor = event.target.result
          if(cursor){
            todoList.push(cursor.value)
            cursor.continue()
          }
        }

        transaction.oncomplete = () => {
          resolve(todoList)
        }

        transaction.onerror = event => {
          reject(event)
        }

      })

    },
    async saveTodo(todo){
      this.database = await this.getDataBase()

      return new Promise((resolve, reject) =>{
        const transaction = this.database.transaction('todos','readwrite')
        const store = transaction.objectStore('todos')

        store.put(todo)

        transaction.oncomplete = () => {
          resolve("El item se agrego correctamente")
        }

        transaction.onerror = event => {
          reject(event)
        }

      })
    },

    async deleteTodo(todo) {
      this.database = await this.getDataBase()
      return new Promise((resolve, reject) => {
        const transaction = this.database.transaction('todos', 'readwrite')
        const store = transaction.objectStore('todos')
        store.delete(todo.id)
        transaction.oncomplete = () => {
          resolve('todo eleiminado')
        }
        transaction.onerror = event => {
          reject(event)
        }
      })
    },

  },
};
</script>

<style>
@import "./styles/todomvc-base.css";
@import "./styles/todomvc-index.css";

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

.btn {
  padding: 0 10px;
}
</style>
