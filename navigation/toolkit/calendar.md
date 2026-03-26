---
title: Calendar
permalink: /student/calendar
layout: aesthetihawk
active_tab: calendar
---
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/fullcalendar@5.11.0/main.min.css">
<link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">

<style>
    .event-tooltip {
        position: absolute;
        background: #1a202c;
        color: white;
        padding: 8px;
        border-radius: 6px;
        font-size: 0.875rem;
        z-index: 10000;
        box-shadow: 0 4px 6px rgba(0,0,0,0.3);
        pointer-events: none;
        max-width: 250px;
    }
</style>

<div id="calendar" class="box-border z-0"></div>

<div id="eventModal" class="fixed z-[99999] inset-0 flex items-center justify-center bg-black bg-opacity-70 backdrop-blur-sm pt-12 hidden">
    <div class="relative bg-gray-900 mx-auto my-12 p-8 rounded-2xl shadow-2xl max-w-xl min-h-fit w-full text-white font-sans modal-content overflow-y-auto max-h-[90vh]">
        <span class="text-gray-400 absolute right-8 top-6 text-3xl font-bold cursor-pointer transition-colors duration-300 hover:text-red-600" id="closeModal">&times;</span>
        <div class="modal-body">
            <h2 id="eventTitle" class="text-white text-4xl font-bold mb-6"></h2>
            
            <label class="block mt-2 mb-1 text-lg font-semibold">Date:</label>
            <p id="editDateDisplay" class="w-full p-3 rounded-xl border border-gray-700 text-base bg-gray-800 text-white mb-4"></p>
            <input type="date" id="editDate" class="w-full p-3 rounded-xl border border-gray-700 text-base bg-gray-800 text-white mb-4 hidden">
            
            <label class="block mt-2 mb-1 text-lg font-semibold">Title:</label>
            <p id="editTitle" contentEditable="false" class="w-full p-3 rounded-xl border border-gray-700 text-base bg-gray-800 text-white mb-4"></p>
            
            <label class="block mt-2 mb-1 text-lg font-semibold">Description:</label>
            <p id="editDescription" contentEditable="false" class="w-full p-3 rounded-xl border border-gray-700 text-base bg-gray-800 text-white mb-4 whitespace-pre-wrap"></p>
            
            <div class="modal-actions space-y-2">
                <button id="saveButton" class="w-full p-3 bg-blue-700 text-white rounded-xl text-base font-bold hover:bg-blue-800 hidden">Save Changes</button>
                <button id="editButton" class="w-full p-3 bg-red-700 text-white rounded-xl text-base font-bold hover:bg-red-900">Edit Event</button>
                <button id="deleteButton" class="w-full p-3 bg-gray-700 text-white rounded-xl text-base font-bold hover:bg-gray-800">Delete Event</button>
            </div>
        </div>
    </div>
</div>

<script src="https://cdn.jsdelivr.net/npm/fullcalendar@5.11.0/main.min.js"></script>
<script type="module">
    import { javaURI, fetchOptions } from '{{site.baseurl}}/assets/js/api/config.js';

    let allEvents = [];
    let currentFilter = null;
    let calendar;
    let currentEvent = null;
    let isAddingNewEvent = false;

    document.addEventListener("DOMContentLoaded", function () {
        const modal = document.getElementById("eventModal");
        
        // Fetch Logic
        async function handleRequest() {
            try {
                const [calendarRes, assignmentRes] = await Promise.all([
                    fetch(`${javaURI}/api/calendar/events`, fetchOptions),
                    fetch(`${javaURI}/api/assignments/`)
                ]);

                const calendarData = calendarRes.ok ? await calendarRes.json() : [];
                const assignmentData = assignmentRes.ok ? await assignmentRes.json() : [];

                allEvents = [];

                calendarData.forEach(event => {
                    let color = event.class === "CSP" ? "#3788d8" : (event.class === "CSSE" ? "#008000" : "#808080");
                    allEvents.push({
                        id: event.id,
                        period: event.period,
                        title: event.title.replace(/\(P[13]\)/gi, ""),
                        description: event.description,
                        start: event.date,
                        color: color
                    });
                });

                assignmentData.forEach(assign => {
                    const [m, d, y] = assign.dueDate.split('/');
                    allEvents.push({
                        id: assign.id,
                        title: assign.name,
                        description: assign.description,
                        start: `${y}-${m.padStart(2, '0')}-${d.padStart(2, '0')}`,
                        color: "#FFA500"
                    });
                });

                displayCalendar(filterEventsByClass(currentFilter));
            } catch (err) {
                console.error("Initialization error:", err);
            }
        }

        function displayCalendar(events) {
            const calendarEl = document.getElementById('calendar');
            if (calendar) calendar.destroy();

            calendar = new FullCalendar.Calendar(calendarEl, {
                initialView: 'dayGridMonth',
                headerToolbar: {
                    left: 'prev,next today allButton,csaButton,cspButton,csseButton',
                    center: 'title',
                    right: 'dayGridMonth,dayGridWeek,dayGridDay'
                },
                customButtons: {
                    allButton: { text: 'All', click: () => { currentFilter = null; displayCalendar(filterEventsByClass(null)); }},
                    csaButton: { text: 'CSA', click: () => { currentFilter = "CSA"; displayCalendar(filterEventsByClass("CSA")); }},
                    cspButton: { text: 'CSP', click: () => { currentFilter = "CSP"; displayCalendar(filterEventsByClass("CSP")); }},
                    csseButton: { text: 'CSSE', click: () => { currentFilter = "CSSE"; displayCalendar(filterEventsByClass("CSSE")); }}
                },
                events: events,
                dateClick: (info) => openModal(null, info.dateStr),
                eventClick: (info) => openModal(info.event)
            });
            calendar.render();
        }

        function openModal(event = null, dateStr = null) {
            currentEvent = event;
            isAddingNewEvent = !event;
            
            const titleEl = document.getElementById("editTitle");
            const descEl = document.getElementById("editDescription");
            const dateInput = document.getElementById("editDate");
            const dateDisplay = document.getElementById("editDateDisplay");

            // Reset States
            titleEl.contentEditable = isAddingNewEvent;
            descEl.contentEditable = isAddingNewEvent;
            dateInput.classList.toggle('hidden', !isAddingNewEvent);
            dateDisplay.classList.toggle('hidden', isAddingNewEvent);
            document.getElementById("saveButton").classList.toggle('hidden', !isAddingNewEvent);
            document.getElementById("editButton").classList.toggle('hidden', isAddingNewEvent);
            document.getElementById("deleteButton").classList.toggle('hidden', isAddingNewEvent);

            if (isAddingNewEvent) {
                document.getElementById("eventTitle").textContent = "Add New Event";
                titleEl.innerHTML = "";
                descEl.innerHTML = "";
                dateInput.value = dateStr;
                dateDisplay.textContent = formatDisplayDate(dateStr);
            } else {
                document.getElementById("eventTitle").textContent = event.title;
                titleEl.innerHTML = event.title;
                descEl.innerHTML = slackToHtml(event.extendedProps.description || "");
                dateDisplay.textContent = formatDisplayDate(event.start);
                dateInput.value = event.startStr.split("T")[0];
            }
            modal.style.display = "flex";
        }

        // Action Handlers
        document.getElementById("saveButton").onclick = async function () {
            const payload = {
                title: document.getElementById("editTitle").innerText.trim(),
                description: document.getElementById("editDescription").innerText,
                date: document.getElementById("editDate").value,
                period: currentFilter
            };

            const url = isAddingNewEvent ? `${javaURI}/api/calendar/add_event` : `${javaURI}/api/calendar/edit/${currentEvent.id}`;
            const method = isAddingNewEvent ? "POST" : "PUT";

            try {
                const res = await fetch(url, {
                    method: method,
                    headers: { "Content-Type": "application/json" },
                    body: JSON.stringify(payload)
                });
                if (res.ok) {
                    handleRequest();
                    closeModalFunc();
                }
            } catch (err) {
                alert("Action failed. Check console.");
            }
        };

        document.getElementById("editButton").onclick = function() {
            document.getElementById("editTitle").contentEditable = true;
            document.getElementById("editDescription").contentEditable = true;
            document.getElementById("editDate").classList.remove('hidden');
            document.getElementById("editDateDisplay").classList.add('hidden');
            this.classList.add('hidden');
            document.getElementById("saveButton").classList.remove('hidden');
        };

        document.getElementById("deleteButton").onclick = async function() {
            if (!confirm("Delete this event?")) return;
            try {
                const res = await fetch(`${javaURI}/api/calendar/delete/${currentEvent.id}`, { method: "DELETE" });
                if (res.ok) {
                    currentEvent.remove();
                    closeModalFunc();
                }
            } catch (err) { console.error(err); }
        };

        function closeModalFunc() {
            modal.style.display = "none";
        }

        document.getElementById("closeModal").onclick = closeModalFunc;
        window.onclick = (e) => { if (e.target === modal) closeModalFunc(); };
        
        handleRequest();
    });

    // Utilities
    function filterEventsByClass(className) {
        return className ? allEvents.filter(e => e.period === className) : allEvents;
    }

    function formatDisplayDate(dateString) {
        const date = new Date(dateString);
        return date.toLocaleDateString('en-US', { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' });
    }

    function slackToHtml(text) {
        if (!text) return '';
        return text
            .replace(/\*([^*]+)\*/g, '<strong>$1</strong>')
            .replace(/_([^_]+)_/g, '<em>$1</em>')
            .replace(/\n/g, '<br>');
    }
</script>