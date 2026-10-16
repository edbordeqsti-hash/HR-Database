<!doctype html>
<html lang="en" class="dark">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>HR Document Portal</title>

  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
</head>

<body class="min-h-screen bg-slate-950 text-slate-100 antialiased">
  <main id="app" class="min-h-screen"></main>

  <script>
    const app = document.getElementById('app');
    const STORAGE_BUCKET = 'employee-docs';
    const REGISTER_FUNCTION = 'hr-create-employee';

    let client = null;
    let currentUser = null;
    let currentProfile = null;
    let adminDocuments = [];
    let employeeDocuments = [];
    let currentPdfUrl = null;

    function createClient(url, key) {
      if (!window.supabase?.createClient) {
        throw new Error('Supabase JavaScript library failed to load.');
      }
      return window.supabase.createClient(url, key);
    }

    function renderErrorScreen(message) {
      app.replaceChildren();

      const wrap = document.createElement('section');
      wrap.className = 'mx-auto max-w-xl p-8';

      const heading = document.createElement('h1');
      heading.className = 'mb-3 text-xl font-bold text-rose-300';
      heading.textContent = 'Something went wrong';

      const text = document.createElement('p');
      text.className = 'mb-5 text-slate-300';
      text.textContent = message;

      const button = document.createElement('button');
      button.className = 'rounded-lg bg-indigo-600 px-4 py-2 font-medium hover:bg-indigo-500';
      button.textContent = 'Sign out';
      button.addEventListener('click', async () => {
        await client?.auth.signOut();
        location.reload();
      });

      wrap.append(heading, text, button);
      app.append(wrap);
    }

    function showConfigScreen() {
      app.innerHTML = `
        <section class="mx-auto flex min-h-screen max-w-md items-center p-6">
          <form id="config-form" class="w-full space-y-4 rounded-2xl border border-slate-700 bg-slate-900 p-6">
            <h1 class="text-2xl font-bold">Connect to Supabase</h1>
            <p class="text-sm text-slate-400">
              Enter your project URL and publishable key. Never enter a secret or service-role key here.
            </p>
            <label class="block text-sm">
              Project URL
              <input id="config-url" required type="url" placeholder="https://your-project.supabase.co"
                class="mt-1 w-full rounded-lg border border-slate-700 bg-slate-950 px-3 py-2">
            </label>
            <label class="block text-sm">
              Publishable key
              <input id="config-key" required type="password" autocomplete="off"
                class="mt-1 w-full rounded-lg border border-slate-700 bg-slate-950 px-3 py-2">
            </label>
            <button class="w-full rounded-lg bg-indigo-600 px-4 py-2 font-semibold hover:bg-indigo-500">
              Connect
            </button>
          </form>
        </section>`;
    }

    function showLoginScreen(errorMessage = '') {
      app.innerHTML = `
        <section class="mx-auto flex min-h-screen max-w-md items-center p-6">
          <form id="login-form" class="w-full space-y-4 rounded-2xl border border-slate-700 bg-slate-900 p-6">
            <div>
              <h1 class="text-2xl font-bold">HR Document Portal</h1>
              <p class="mt-1 text-sm text-slate-400">DTR and payslip management</p>
            </div>
            <p id="login-error" class="hidden rounded-lg bg-rose-950 p-3 text-sm text-rose-200"></p>
            <label class="block text-sm">
              Email
              <input id="login-email" required type="email" autocomplete="username"
                class="mt-1 w-full rounded-lg border border-slate-700 bg-slate-950 px-3 py-2">
            </label>
            <label class="block text-sm">
              Password
              <input id="login-password" required type="password" autocomplete="current-password"
                class="mt-1 w-full rounded-lg border border-slate-700 bg-slate-950 px-3 py-2">
            </label>
            <button class="w-full rounded-lg bg-indigo-600 px-4 py-2 font-semibold hover:bg-indigo-500">
              Sign in
            </button>
          </form>
        </section>`;

      if (errorMessage) {
        const error = document.getElementById('login-error');
        error.textContent = errorMessage;
        error.classList.remove('hidden');
      }
    }

    function makeButton(label, action, extraClass = '') {
      const button = document.createElement('button');
      button.type = 'button';
      button.dataset.action = action;
      button.textContent = label;
      button.className =
        `rounded-lg border border-slate-700 px-3 py-1.5 text-sm hover:bg-slate-800 ${extraClass}`;
      return button;
    }

    function makeCell(text, className = 'p-3') {
      const cell = document.createElement('td');
      cell.className = className;
      cell.textContent = text == null ? '' : String(text);
      return cell;
    }

    function makeTableRow(values) {
      const row = document.createElement('tr');
      row.className = 'border-t border-slate-800 hover:bg-slate-900';
      for (const value of values) row.append(makeCell(value));
      return row;
    }

    function makeDocumentActions(doc) {
      const cell = document.createElement('td');
      cell.className = 'space-x-2 p-3 text-right';

      const view = makeButton('View', 'view-document', 'bg-indigo-600 text-white');
      view.dataset.path = doc.file_path || '';
      view.dataset.name = doc.file_name || 'Document.pdf';

      const download = makeButton('Download', 'download-document');
      download.dataset.path = doc.file_path || '';
      download.dataset.name = doc.file_name || 'Document.pdf';

      cell.append(view, download);
      return cell;
    }

    async function signOut() {
      const { error } = await client.auth.signOut();
      if (error) alert(error.message);
      location.reload();
    }

    function renderHeader(title, subtitle) {
      const header = document.createElement('header');
      header.className = 'flex items-center justify-between border-b border-slate-800 bg-slate-900 px-6 py-4';

      const left = document.createElement('div');
      const h1 = document.createElement('h1');
      h1.className = 'font-bold';
      h1.textContent = title;

      const p = document.createElement('p');
      p.className = 'text-sm text-slate-400';
      p.textContent = subtitle;

      left.append(h1, p);

      const logout = makeButton('Sign out', 'sign-out');
      header.append(left, logout);
      return header;
    }

    function renderAdminShell() {
      app.replaceChildren();
      app.append(renderHeader('HR Admin Dashboard', `Welcome, ${currentProfile.full_name || ''}`));

      app.insertAdjacentHTML('beforeend', `
        <section class="mx-auto max-w-7xl space-y-6 p-6">
          <nav class="flex flex-wrap gap-2 border-b border-slate-800 pb-3">
            <button data-action="switch-tab" data-tab="documents" class="rounded-lg bg-indigo-600 px-4 py-2 text-sm">Documents</button>
            <button data-action="switch-tab" data-tab="employees" class="rounded-lg px-4 py-2 text-sm text-slate-300 hover:bg-slate-800">Employees</button>
            <button data-action="switch-tab" data-tab="logs" class="rounded-lg px-4 py-2 text-sm text-slate-300 hover:bg-slate-800">Audit logs</button>
          </nav>

          <div id="admin-error" class="hidden rounded-lg bg-rose-950 p-3 text-sm text-rose-200"></div>

          <section id="tab-documents" class="space-y-4">
            <div class="grid gap-6 lg:grid-cols-3">
              <form id="upload-form" class="space-y-4 rounded-2xl border border-slate-800 bg-slate-900 p-5">
                <h2 class="text-lg font-bold">Upload a document</h2>
                <label class="block text-sm">Employee
                  <select id="upload-employee" required class="mt-1 w-full rounded-lg border border-slate-700 bg-slate-950 p-2"></select>
                </label>
                <label class="block text-sm">Type
                  <select id="upload-type" class="mt-1 w-full rounded-lg border border-slate-700 bg-slate-950 p-2">
                    <option value="dtr">Daily Time Record (DTR)</option>
                    <option value="payslip">Payslip</option>
                  </select>
                </label>
                <label class="block text-sm">Payroll period
                  <input id="upload-period" required class="mt-1 w-full rounded-lg border border-slate-700 bg-slate-950 p-2">
                </label>
                <label class="block text-sm">PDF file
                  <input id="upload-file" required type="file" accept="application/pdf"
                    class="mt-1 block w-full text-sm">
                </label>
                <label class="flex items-center gap-2 text-sm">
                  <input id="upload-replace" type="checkbox">
                  Replace matching employee/type/period document
                </label>
                <button class="w-full rounded-lg bg-indigo-600 px-4 py-2 font-semibold hover:bg-indigo-500">Upload</button>
              </form>

              <div class="overflow-x-auto rounded-2xl border border-slate-800 bg-slate-900 p-5 lg:col-span-2">
                <div class="mb-4 flex flex-wrap items-center justify-between gap-3">
                  <h2 class="text-lg font-bold">Documents</h2>
                  <input id="admin-search" placeholder="Search documents"
                    class="rounded-lg border border-slate-700 bg-slate-950 px-3 py-2 text-sm">
                </div>
                <table class="w-full text-left text-sm">
                  <thead><tr class="text-slate-400">
                    <th class="p-3">Employee</th><th class="p-3">Type</th>
                    <th class="p-3">Period</th><th class="p-3">Status</th>
                    <th class="p-3 text-right">Actions</th>
                  </tr></thead>
                  <tbody id="admin-documents"></tbody>
                </table>
              </div>
            </div>
          </section>

          <section id="tab-employees" class="hidden space-y-4">
            <div class="grid gap-6 lg:grid-cols-3">
              <form id="employee-form" class="space-y-4 rounded-2xl border border-slate-800 bg-slate-900 p-5">
                <h2 class="text-lg font-bold">Invite employee</h2>
                <p class="text-sm text-slate-400">The server-side function creates the account and sends the invitation.</p>
                <label class="block text-sm">Employee ID
                  <input id="employee-id" required class="mt-1 w-full rounded-lg border border-slate-700 bg-slate-950 p-2">
                </label>
                <label class="block text-sm">Full name
                  <input id="employee-name" required class="mt-1 w-full rounded-lg border border-slate-700 bg-slate-950 p-2">
                </label>
                <label class="block text-sm">Email
                  <input id="employee-email" required type="email" class="mt-1 w-full rounded-lg border border-slate-700 bg-slate-950 p-2">
                </label>
                <button class="w-full rounded-lg bg-indigo-600 px-4 py-2 font-semibold hover:bg-indigo-500">Send invitation</button>
              </form>
              <div class="overflow-x-auto rounded-2xl border border-slate-800 bg-slate-900 p-5 lg:col-span-2">
                <h2 class="mb-4 text-lg font-bold">Employee directory</h2>
                <table class="w-full text-left text-sm">
                  <thead><tr class="text-slate-400">
                    <th class="p-3">Employee ID</th><th class="p-3">Name</th>
                    <th class="p-3">Profile ID</th><th class="p-3">Role</th>
                  </tr></thead>
                  <tbody id="employee-list"></tbody>
                </table>
              </div>
            </div>
          </section>

          <section id="tab-logs" class="hidden overflow-x-auto rounded-2xl border border-slate-800 bg-slate-900 p-5">
            <h2 class="mb-4 text-lg font-bold">Recent audit logs</h2>
            <table class="w-full text-left text-sm">
              <thead><tr class="text-slate-400">
                <th class="p-3">Timestamp</th><th class="p-3">User ID</th>
                <th class="p-3">Action</th><th class="p-3">Details</th>
              </tr></thead>
              <tbody id="audit-list"></tbody>
            </table>
          </section>
        </section>`);
    }

    function renderEmployeeShell() {
      app.replaceChildren();
      app.append(renderHeader(
        'Employee Portal',
        `${currentProfile.full_name || ''} (${currentProfile.employee_id || ''})`
      ));

      app.insertAdjacentHTML('beforeend', `
        <section class="mx-auto max-w-6xl p-6">
          <div class="overflow-x-auto rounded-2xl border border-slate-800 bg-slate-900 p-5">
            <div class="mb-4 flex flex-wrap items-center justify-between gap-3">
              <div>
                <h2 class="text-xl font-bold">My DTRs and payslips</h2>
                <p class="text-sm text-slate-400">Your assigned documents.</p>
              </div>
              <input id="employee-search" placeholder="Search period or type"
                class="rounded-lg border border-slate-700 bg-slate-950 px-3 py-2 text-sm">
            </div>
            <table class="w-full text-left text-sm">
              <thead><tr class="text-slate-400">
                <th class="p-3">Type</th><th class="p-3">Period</th>
                <th class="p-3">Status</th><th class="p-3">Uploaded</th>
                <th class="p-3 text-right">Actions</th>
              </tr></thead>
              <tbody id="employee-documents"></tbody>
            </table>
          </div>
        </section>`);
    }

    function renderAdminDocuments(rows) {
      const tbody = document.getElementById('admin-documents');
      if (!tbody) return;
      tbody.replaceChildren();

      if (!rows.length) {
        tbody.append(makeTableRow(['No documents found.', '', '', '', '']));
        return;
      }

      for (const doc of rows) {
        const row = document.createElement('tr');
        row.className = 'border-t border-slate-800 hover:bg-slate-900';
        row.append(
          makeCell(doc.profiles?.full_name || 'Unknown'),
          makeCell(doc.doc_type),
          makeCell(doc.payroll_period),
          makeCell(doc.verification_status || 'pending')
        );
        row.append(makeDocumentActions(doc));
        tbody.append(row);
      }
    }

    function renderEmployeeDocuments(rows) {
      const tbody = document.getElementById('employee-documents');
      if (!tbody) return;
      tbody.replaceChildren();

      if (!rows.length) {
        tbody.append(makeTableRow(['No documents assigned to you.', '', '', '', '']));
        return;
      }

      for (const doc of rows) {
        const row = document.createElement('tr');
        row.className = 'border-t border-slate-800 hover:bg-slate-900';
        let date = '';
        if (doc.created_at) {
          const parsed = new Date(doc.created_at);
          if (!Number.isNaN(parsed.valueOf())) date = parsed.toLocaleDateString();
        }
        row.append(
          makeCell(doc.doc_type),
          makeCell(doc.payroll_period),
          makeCell(doc.verification_status || 'pending'),
          makeCell(date)
        );
        row.append(makeDocumentActions(doc));
        tbody.append(row);
      }
    }

    async function loadAdminData() {
      const [profilesResult, docsResult, logsResult] = await Promise.all([
        client.from('profiles').select('id, employee_id, full_name, role').order('full_name'),
        client.from('documents')
          .select('*, profiles(full_name, employee_id)')
          .order('created_at', { ascending: false }),
        client.from('audit_logs').select('created_at, user_id, action, details')
          .order('created_at', { ascending: false }).limit(50)
      ]);

      for (const result of [profilesResult, docsResult, logsResult]) {
        if (result.error) throw result.error;
      }

      const profiles = profilesResult.data || [];
      adminDocuments = docsResult.data || [];

      const select = document.getElementById('upload-employee');
      select.replaceChildren();
      for (const profile of profiles.filter(p => p.role !== 'hr')) {
        const option = document.createElement('option');
        option.value = profile.id;
        option.textContent = `${profile.full_name || 'Unnamed'} (${profile.employee_id || 'No employee ID'})`;
        select.append(option);
      }

      const employeeList = document.getElementById('employee-list');
      employeeList.replaceChildren();
      for (const profile of profiles) {
        employeeList.append(makeTableRow([
          profile.employee_id,
          profile.full_name,
          profile.id,
          profile.role
        ]));
      }

      const auditList = document.getElementById('audit-list');
      auditList.replaceChildren();
      for (const log of logsResult.data || []) {
        const date = log.created_at ? new Date(log.created_at).toLocaleString() : '';
        auditList.append(makeTableRow([date, log.user_id, log.action, log.details]));
      }

      renderAdminDocuments(adminDocuments);
    }

    async function loadEmployeeDocuments() {
      const { data, error } = await client
        .from('documents')
        .select('*')
        .eq('employee_id', currentProfile.id)
        .order('created_at', { ascending: false });

      if (error) throw error;
      employeeDocuments = data || [];
      renderEmployeeDocuments(employeeDocuments);
    }

    function switchAdminTab(tabName) {
      for (const name of ['documents', 'employees', 'logs']) {
        document.getElementById(`tab-${name}`).classList.toggle('hidden', name !== tabName);
      }
      document.querySelectorAll('[data-action="switch-tab"]').forEach(button => {
        button.classList.toggle('bg-indigo-600', button.dataset.tab === tabName);
      });
    }

    async function uploadDocument(event) {
      event.preventDefault();

      const employeeId = document.getElementById('upload-employee').value;
      const docType = document.getElementById('upload-type').value;
      const period = document.getElementById('upload-period').value.trim();
      const file = document.getElementById('upload-file').files[0];
      const replace = document.getElementById('upload-replace').checked;

      if (!file || file.type !== 'application/pdf') {
        alert('Choose a PDF file.');
        return;
      }
      if (file.size > 10 * 1024 * 1024) {
        alert('The PDF must be 10 MB or smaller.');
        return;
      }

      const existing = adminDocuments.find(doc =>
        doc.employee_id === employeeId &&
        doc.doc_type === docType &&
        doc.payroll_period === period
      );

      if (existing && !replace) {
        alert('A document already exists for this employee, type, and period.');
        return;
      }

      const safeFileName = file.name.replace(/[^a-zA-Z0-9._-]/g, '_');
      const filePath = `${employeeId}/${crypto.randomUUID()}_${safeFileName}`;

      const { error: uploadError } = await client.storage
        .from(STORAGE_BUCKET)
        .upload(filePath, file, { contentType: 'application/pdf', upsert: false });

      if (uploadError) throw uploadError;

      const { error: insertError } = await client.from('documents').insert({
        employee_id: employeeId,
        doc_type: docType,
        payroll_period: period,
        file_path: filePath,
        file_name: file.name,
        file_size: file.size,
        uploaded_by: currentProfile.id
      });

      if (insertError) {
        await client.storage.from(STORAGE_BUCKET).remove([filePath]);
        throw insertError;
      }

      if (existing && replace) {
        const { error: deleteRowError } = await client.from('documents')
          .delete().eq('id', existing.id);
        if (!deleteRowError) {
          await client.storage.from(STORAGE_BUCKET).remove([existing.file_path]);
        }
      }

      await client.from('audit_logs').insert({
        user_id: currentProfile.id,
        action: existing && replace ? 'REPLACE' : 'UPLOAD',
        details: `Uploaded ${docType.toUpperCase()} for ${period}`
      });

      document.getElementById('upload-form').reset();
      await loadAdminData();
      alert('Document uploaded.');
    }

    async function inviteEmployee(event) {
      event.preventDefault();

      const employee_id = document.getElementById('employee-id').value.trim();
      const full_name = document.getElementById('employee-name').value.trim();
      const email = document.getElementById('employee-email').value.trim();

      const { error } = await client.functions.invoke(REGISTER_FUNCTION, {
        body: { employee_id, full_name, email }
      });

      if (error) throw error;

      document.getElementById('employee-form').reset();
      await loadAdminData();
      alert('Employee invitation sent.');
    }

    async function openDocument(path, name, shouldView) {
      const { data, error } = await client.storage
        .from(STORAGE_BUCKET)
        .createSignedUrl(path, 120);

      if (error) throw error;

      await client.from('audit_logs').insert({
        user_id: currentProfile.id,
        action: shouldView ? 'VIEW' : 'DOWNLOAD',
        details: `${shouldView ? 'Viewed' : 'Downloaded'} document: ${name}`
      });

      if (shouldView) {
        currentPdfUrl = data.signedUrl;
        showPdfModal(name, data.signedUrl);
      } else {
        const link = document.createElement('a');
        link.href = data.signedUrl;
        link.download = name;
        link.rel = 'noopener';
        document.body.append(link);
        link.click();
        link.remove();
      }
    }

    function showPdfModal(title, url) {
      const modal = document.createElement('div');
      modal.id = 'pdf-modal';
      modal.className = 'fixed inset-0 z-50 flex flex-col bg-black/90 p-4';

      const bar = document.createElement('div');
      bar.className = 'mb-3 flex items-center justify-between';
      const heading = document.createElement('h2');
      heading.className = 'truncate font-semibold';
      heading.textContent = title;

      const close = makeButton('Close', 'close-pdf', 'bg-rose-700 text-white');
      bar.append(heading, close);

      const frame = document.createElement('iframe');
      frame.title = 'PDF document';
      frame.src = url;
      frame.className = 'min-h-0 flex-1 rounded bg-white';

      modal.append(bar, frame);
      document.body.append(modal);
    }

    function filterRows(rows, query) {
      const needle = String(query || '').trim().toLowerCase();
      if (!needle) return rows;

      return rows.filter(row => [
        row.doc_type,
        row.payroll_period,
        row.profiles?.full_name,
        row.profiles?.employee_id
      ].some(value => String(value ?? '').toLowerCase().includes(needle)));
    }

    document.addEventListener('submit', async event => {
      try {
        if (event.target.id === 'config-form') {
          event.preventDefault();
          const url = document.getElementById('config-url').value.trim();
          const key = document.getElementById('config-key').value.trim();

          if (!url.startsWith('https://')) throw new Error('Use your HTTPS Supabase project URL.');

          localStorage.setItem('sb_url', url);
          localStorage.setItem('sb_key', key);
          location.reload();
        }

        if (event.target.id === 'login-form') {
          event.preventDefault();
          const { error } = await client.auth.signInWithPassword({
            email: document.getElementById('login-email').value.trim(),
            password: document.getElementById('login-password').value
          });
          if (error) showLoginScreen(error.message);
          else await initializeApp();
        }

        if (event.target.id === 'upload-form') await uploadDocument(event);
        if (event.target.id === 'employee-form') await inviteEmployee(event);
      } catch (error) {
        alert(error?.message || 'Request failed.');
      }
    });

    document.addEventListener('click', async event => {
      const button = event.target.closest('[data-action]');
      if (!button) return;

      try {
        switch (button.dataset.action) {
          case 'sign-out':
            await signOut();
            break;
          case 'switch-tab':
            switchAdminTab(button.dataset.tab);
            break;
          case 'view-document':
            await openDocument(button.dataset.path, button.dataset.name, true);
            break;
          case 'download-document':
            await openDocument(button.dataset.path, button.dataset.name, false);
            break;
          case 'close-pdf':
            document.getElementById('pdf-modal')?.remove();
            currentPdfUrl = null;
            break;
        }
      } catch (error) {
        alert(error?.message || 'Request failed.');
      }
    });

    document.addEventListener('input', event => {
      if (event.target.id === 'admin-search') {
        renderAdminDocuments(filterRows(adminDocuments, event.target.value));
      }
      if (event.target.id === 'employee-search') {
        const query = event.target.value.trim().toLowerCase();
        const rows = employeeDocuments.filter(doc =>
          [doc.doc_type, doc.payroll_period].some(value =>
            String(value ?? '').toLowerCase().includes(query)
          )
        );
        renderEmployeeDocuments(rows);
      }
    });

    async function initializeApp() {
      try {
        const { data: { session }, error: sessionError } = await client.auth.getSession();
        if (sessionError) throw sessionError;

        if (!session) {
          showLoginScreen();
          return;
        }

        currentUser = session.user;

        const { data: profile, error } = await client
          .from('profiles')
          .select('id, employee_id, full_name, role')
          .eq('id', currentUser.id)
          .single();

        if (error || !profile) {
          renderErrorScreen('Your profile could not be loaded. Ask an administrator to check your profile and access policies.');
          return;
        }

        currentProfile = profile;

        if (profile.role === 'hr') {
          renderAdminShell();
          await loadAdminData();
        } else {
          renderEmployeeShell();
          await loadEmployeeDocuments();
        }
      } catch (error) {
        renderErrorScreen(error?.message || 'The app could not be initialized.');
      }
    }

    (async function boot() {
      const url = localStorage.getItem('sb_url');
      const key = localStorage.getItem('sb_key');

      if (!url || !key) {
        showConfigScreen();
        return;
      }

      try {
        client = createClient(url, key);
        await initializeApp();
      } catch (error) {
        renderErrorScreen(error?.message || 'Could not connect to Supabase.');
      }
    })();
  </script>
</body>
</html>
