<script setup lang="ts">
import { Head } from '@inertiajs/vue3';
import { CheckCircle2, Clipboard, Copy, Database, HardDrive, RotateCcw, Server, Terminal, Zap } from '@lucide/vue';
import { computed, ref, watch } from 'vue';

type OsKey = 'macos' | 'linux' | 'wsl';
type ServiceKey = 'web' | 'database' | 'runtime' | 'queues';
type ServiceStatus = 'healthy' | 'failing';

type Fix = {
    id: string;
    label: string;
    cause: string;
    commands: Record<OsKey, string>;
};

type Service = {
    id: ServiceKey;
    name: string;
    shortName: string;
    description: string;
    ports: string;
    Icon: typeof Server;
    fixes: Fix[];
};

const operatingSystems: { id: OsKey; label: string; hint: string }[] = [
    { id: 'macos', label: 'macOS', hint: 'Homebrew-first commands' },
    { id: 'linux', label: 'Linux (Ubuntu/Debian)', hint: 'systemd + apt paths' },
    { id: 'wsl', label: 'Windows (WSL)', hint: 'WSL local dev shell' },
];

const services: Service[] = [
    {
        id: 'web',
        name: 'Web Server',
        shortName: 'Nginx / Apache',
        description: 'HTTP listener, virtual hosts, local TLS, and reverse proxies.',
        ports: '80, 443, 8080',
        Icon: Server,
        fixes: [
            {
                id: 'port-80',
                label: 'Port 80 Already in Use',
                cause: 'This usually happens when another local web server is already bound to the HTTP port.',
                commands: {
                    macos: `sudo lsof -nP -iTCP:80 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:80)
brew services restart nginx || brew services restart httpd`,
                    linux: `sudo lsof -nP -iTCP:80 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:80)
sudo systemctl restart nginx || sudo systemctl restart apache2`,
                    wsl: `sudo lsof -nP -iTCP:80 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:80)
sudo service nginx restart || sudo service apache2 restart`,
                },
            },
            {
                id: 'bad-vhost',
                label: 'Site Loads Default Page',
                cause: 'Your web server is running, but the active virtual host is not pointing at the project document root.',
                commands: {
                    macos: `nginx -t
sudo sed -n '1,160p' /opt/homebrew/etc/nginx/servers/*.conf
brew services restart nginx`,
                    linux: `sudo nginx -t
sudo a2ensite your-project.conf
sudo systemctl reload nginx || sudo systemctl reload apache2`,
                    wsl: `sudo nginx -t
sudo a2ensite your-project.conf
sudo service nginx reload || sudo service apache2 reload`,
                },
            },
            {
                id: 'permission-denied',
                label: '403 Forbidden / Permission Issue',
                cause: 'The web server user can read the server config but cannot access the project files.',
                commands: {
                    macos: `chmod -R u+rwX,go+rX .
find . -type d -exec chmod 755 {} \\;
find . -type f -exec chmod 644 {} \\;`,
                    linux: `sudo chown -R $USER:www-data .
sudo find . -type d -exec chmod 775 {} \\;
sudo find . -type f -exec chmod 664 {} \\;`,
                    wsl: `sudo chown -R $USER:www-data .
sudo find . -type d -exec chmod 775 {} \\;
sudo find . -type f -exec chmod 664 {} \\;`,
                },
            },
        ],
    },
    {
        id: 'database',
        name: 'Database',
        shortName: 'MySQL / Postgres',
        description: 'Local SQL credentials, ports, sockets, and service health.',
        ports: '3306, 5432',
        Icon: Database,
        fixes: [
            {
                id: 'access-denied',
                label: 'Access Denied / Password Issue',
                cause: 'The application credentials do not match a local database user with the expected privileges.',
                commands: {
                    macos: `mysql -uroot -p -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'password'; FLUSH PRIVILEGES;"
grep -n "DB_" .env
php artisan config:clear 2>/dev/null || true`,
                    linux: `sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password'; FLUSH PRIVILEGES;"
grep -n "DB_" .env
php artisan config:clear 2>/dev/null || true`,
                    wsl: `sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password'; FLUSH PRIVILEGES;"
grep -n "DB_" .env
php artisan config:clear 2>/dev/null || true`,
                },
            },
            {
                id: 'db-port',
                label: 'Port 3306 Already in Use',
                cause: 'A previous database process or another local stack is occupying the MySQL port.',
                commands: {
                    macos: `sudo lsof -nP -iTCP:3306 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:3306)
brew services restart mysql`,
                    linux: `sudo lsof -nP -iTCP:3306 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:3306)
sudo systemctl restart mysql`,
                    wsl: `sudo lsof -nP -iTCP:3306 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:3306)
sudo service mysql restart`,
                },
            },
            {
                id: 'missing-db',
                label: 'Unknown Database / Missing Schema',
                cause: 'The application can connect to SQL, but the configured database has not been created locally.',
                commands: {
                    macos: `createdb dev_database 2>/dev/null || mysql -uroot -p -e "CREATE DATABASE IF NOT EXISTS dev_database;"
php artisan migrate 2>/dev/null || npm run migrate`,
                    linux: `createdb dev_database 2>/dev/null || mysql -uroot -p -e "CREATE DATABASE IF NOT EXISTS dev_database;"
php artisan migrate 2>/dev/null || npm run migrate`,
                    wsl: `createdb dev_database 2>/dev/null || mysql -uroot -p -e "CREATE DATABASE IF NOT EXISTS dev_database;"
php artisan migrate 2>/dev/null || npm run migrate`,
                },
            },
        ],
    },
    {
        id: 'runtime',
        name: 'Runtime',
        shortName: 'Node / PHP / Python',
        description: 'Language version managers, packages, caches, and dev servers.',
        ports: '3000, 5173, 8000',
        Icon: Zap,
        fixes: [
            {
                id: 'wrong-version',
                label: 'Wrong Runtime Version',
                cause: 'Your shell is using a different language runtime than the project expects.',
                commands: {
                    macos: `node -v && php -v && python3 --version
nvm use 2>/dev/null || fnm use 2>/dev/null || true
composer install && npm install`,
                    linux: `node -v && php -v && python3 --version
nvm use 2>/dev/null || fnm use 2>/dev/null || true
composer install && npm install`,
                    wsl: `node -v && php -v && python3 --version
nvm use 2>/dev/null || fnm use 2>/dev/null || true
composer install && npm install`,
                },
            },
            {
                id: 'node-modules',
                label: 'Module Not Found / Broken Install',
                cause: 'Dependency metadata and installed packages have drifted out of sync.',
                commands: {
                    macos: `rm -rf node_modules package-lock.json
npm cache verify
npm install
npm run dev`,
                    linux: `rm -rf node_modules package-lock.json
npm cache verify
npm install
npm run dev`,
                    wsl: `rm -rf node_modules package-lock.json
npm cache verify
npm install
npm run dev`,
                },
            },
            {
                id: 'vite-port',
                label: 'Dev Server Port Blocked',
                cause: 'The frontend dev server cannot start because its expected port is already occupied.',
                commands: {
                    macos: `sudo lsof -nP -iTCP:5173 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:5173)
npm run dev -- --host 127.0.0.1`,
                    linux: `sudo lsof -nP -iTCP:5173 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:5173)
npm run dev -- --host 127.0.0.1`,
                    wsl: `sudo lsof -nP -iTCP:5173 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:5173)
npm run dev -- --host 0.0.0.0`,
                },
            },
        ],
    },
    {
        id: 'queues',
        name: 'Background Queues',
        shortName: 'Redis / RabbitMQ',
        description: 'Async workers, cache stores, broker ports, and stale jobs.',
        ports: '6379, 5672',
        Icon: HardDrive,
        fixes: [
            {
                id: 'redis-refused',
                label: 'Redis Connection Refused',
                cause: 'The app is trying to use Redis, but the local Redis service is stopped or unreachable.',
                commands: {
                    macos: `brew services restart redis
redis-cli ping
php artisan queue:restart 2>/dev/null || true`,
                    linux: `sudo systemctl restart redis-server
redis-cli ping
php artisan queue:restart 2>/dev/null || true`,
                    wsl: `sudo service redis-server restart
redis-cli ping
php artisan queue:restart 2>/dev/null || true`,
                },
            },
            {
                id: 'stuck-jobs',
                label: 'Jobs Stuck / Worker Not Processing',
                cause: 'Queue workers are stale, stopped, or still running old application code.',
                commands: {
                    macos: `php artisan queue:restart 2>/dev/null || true
pkill -f "queue:work" || true
php artisan queue:work --tries=3`,
                    linux: `php artisan queue:restart 2>/dev/null || true
pkill -f "queue:work" || true
php artisan queue:work --tries=3`,
                    wsl: `php artisan queue:restart 2>/dev/null || true
pkill -f "queue:work" || true
php artisan queue:work --tries=3`,
                },
            },
            {
                id: 'rabbit-port',
                label: 'RabbitMQ Port 5672 Already in Use',
                cause: 'A local broker or stale RabbitMQ process is already listening on the AMQP port.',
                commands: {
                    macos: `sudo lsof -nP -iTCP:5672 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:5672)
brew services restart rabbitmq`,
                    linux: `sudo lsof -nP -iTCP:5672 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:5672)
sudo systemctl restart rabbitmq-server`,
                    wsl: `sudo lsof -nP -iTCP:5672 -sTCP:LISTEN
sudo kill -9 $(sudo lsof -t -i:5672)
sudo service rabbitmq-server restart`,
                },
            },
        ],
    },
];

const selectedOs = ref<OsKey>('linux');
const selectedService = ref<ServiceKey>('web');
const selectedSymptom = ref(services[0].fixes[0].id);
const copied = ref(false);
const statuses = ref<Record<ServiceKey, ServiceStatus>>({
    web: 'failing',
    database: 'healthy',
    runtime: 'healthy',
    queues: 'healthy',
});

const activeService = computed(() => services.find((service) => service.id === selectedService.value) ?? services[0]);
const activeFix = computed(() => activeService.value.fixes.find((fix) => fix.id === selectedSymptom.value) ?? activeService.value.fixes[0]);
const activeOs = computed(() => operatingSystems.find((os) => os.id === selectedOs.value) ?? operatingSystems[0]);
const commandText = computed(() => activeFix.value.commands[selectedOs.value]);
const markdownOutput = computed(() => `${activeFix.value.cause}\n\n\`\`\`bash\n${commandText.value}\n\`\`\``);

watch(selectedService, () => {
    selectedSymptom.value = activeService.value.fixes[0].id;
    statuses.value[selectedService.value] = 'failing';
});

watch([selectedOs, selectedService, selectedSymptom], () => {
    copied.value = false;
});

function selectService(serviceId: ServiceKey) {
    selectedService.value = serviceId;
}

function toggleStatus(serviceId: ServiceKey) {
    statuses.value[serviceId] = statuses.value[serviceId] === 'healthy' ? 'failing' : 'healthy';
    selectedService.value = serviceId;
}

async function copyCommands() {
    await navigator.clipboard.writeText(commandText.value);
    copied.value = true;
    window.setTimeout(() => {
        copied.value = false;
    }, 1800);
}
</script>

<template>
    <Head title="DevStrait" />

    <main class="min-h-screen bg-zinc-950 text-zinc-100">
        <div class="mx-auto flex min-h-screen w-full max-w-7xl flex-col gap-5 px-4 py-5 sm:px-6 lg:px-8">
            <header class="flex flex-col gap-4 border-b border-white/10 pb-5 lg:flex-row lg:items-end lg:justify-between">
                <div>
                    <div class="flex items-center gap-3">
                        <span class="flex size-10 items-center justify-center rounded-md border border-emerald-400/30 bg-emerald-400/10 text-emerald-300">
                            <Terminal class="size-5" />
                        </span>
                        <div>
                            <p class="text-xs font-semibold uppercase tracking-[0.24em] text-emerald-300">DevStrait Core</p>
                            <h1 class="text-3xl font-semibold text-white sm:text-4xl">DevStrait</h1>
                        </div>
                    </div>
                    <p class="mt-3 max-w-2xl text-sm leading-6 text-zinc-400">
                        Local Dev Environment Debugger & Command Builder for web servers, databases, runtimes, and queue workers.
                    </p>
                </div>

                <div class="grid gap-2 rounded-md border border-white/10 bg-zinc-900/80 p-1 sm:grid-cols-3">
                    <button
                        v-for="os in operatingSystems"
                        :key="os.id"
                        type="button"
                        class="rounded px-4 py-3 text-left transition duration-200"
                        :class="selectedOs === os.id ? 'bg-emerald-400 text-zinc-950 shadow-lg shadow-emerald-950/40' : 'text-zinc-300 hover:bg-white/5 hover:text-white'"
                        @click="selectedOs = os.id"
                    >
                        <span class="block text-sm font-semibold">{{ os.label }}</span>
                        <span class="mt-1 block text-xs" :class="selectedOs === os.id ? 'text-zinc-800' : 'text-zinc-500'">{{ os.hint }}</span>
                    </button>
                </div>
            </header>

            <section class="grid flex-1 gap-5 lg:grid-cols-[minmax(0,0.95fr)_minmax(480px,1.05fr)]">
                <aside class="space-y-5">
                    <div class="rounded-lg border border-white/10 bg-zinc-900/70 p-4 shadow-2xl shadow-black/30">
                        <div class="mb-4 flex items-center justify-between gap-3">
                            <div>
                                <h2 class="text-lg font-semibold text-white">Problem Selector</h2>
                                <p class="text-sm text-zinc-400">Mark a failing service, then match the symptom.</p>
                            </div>
                            <button
                                type="button"
                                class="inline-flex size-9 items-center justify-center rounded-md border border-white/10 text-zinc-400 transition hover:border-amber-300/40 hover:bg-amber-300/10 hover:text-amber-200"
                                title="Reset statuses"
                                @click="statuses = { web: 'failing', database: 'healthy', runtime: 'healthy', queues: 'healthy' }"
                            >
                                <RotateCcw class="size-4" />
                            </button>
                        </div>

                        <div class="grid gap-3 sm:grid-cols-2">
                            <button
                                v-for="service in services"
                                :key="service.id"
                                type="button"
                                class="group rounded-lg border p-4 text-left transition duration-200"
                                :class="[
                                    selectedService === service.id ? 'border-emerald-300/70 bg-emerald-400/10 shadow-lg shadow-emerald-950/30' : 'border-white/10 bg-zinc-950/50 hover:border-white/20 hover:bg-zinc-900',
                                    statuses[service.id] === 'failing' ? 'ring-1 ring-rose-400/30' : 'ring-1 ring-emerald-400/20',
                                ]"
                                @click="selectService(service.id)"
                            >
                                <div class="flex items-start justify-between gap-3">
                                    <span
                                        class="flex size-10 items-center justify-center rounded-md border transition"
                                        :class="statuses[service.id] === 'failing' ? 'border-rose-400/30 bg-rose-400/10 text-rose-300' : 'border-emerald-400/30 bg-emerald-400/10 text-emerald-300'"
                                    >
                                        <component :is="service.Icon" class="size-5" />
                                    </span>
                                    <span
                                        class="rounded-full px-2.5 py-1 text-xs font-semibold"
                                        :class="statuses[service.id] === 'failing' ? 'bg-rose-400/10 text-rose-300' : 'bg-emerald-400/10 text-emerald-300'"
                                    >
                                        {{ statuses[service.id] === 'failing' ? 'Failing' : 'Healthy' }}
                                    </span>
                                </div>
                                <h3 class="mt-4 font-semibold text-white">{{ service.name }}</h3>
                                <p class="text-sm text-zinc-400">{{ service.shortName }}</p>
                                <p class="mt-3 min-h-12 text-xs leading-5 text-zinc-500">{{ service.description }}</p>
                                <div class="mt-4 flex items-center justify-between gap-3 border-t border-white/10 pt-3">
                                    <span class="font-mono text-xs text-zinc-500">ports {{ service.ports }}</span>
                                    <span
                                        class="relative h-6 w-11 rounded-full transition"
                                        :class="statuses[service.id] === 'failing' ? 'bg-rose-500/80' : 'bg-emerald-500/80'"
                                        role="switch"
                                        :aria-checked="statuses[service.id] === 'healthy'"
                                        @click.stop="toggleStatus(service.id)"
                                    >
                                        <span
                                            class="absolute top-1 size-4 rounded-full bg-white transition"
                                            :class="statuses[service.id] === 'failing' ? 'left-1' : 'left-6'"
                                        />
                                    </span>
                                </div>
                            </button>
                        </div>
                    </div>

                    <div class="rounded-lg border border-white/10 bg-zinc-900/70 p-4">
                        <label for="symptom" class="text-sm font-semibold text-white">Symptom Matcher</label>
                        <p class="mt-1 text-sm text-zinc-400">Filtered for {{ activeService.name }} on {{ activeOs.label }}.</p>
                        <select
                            id="symptom"
                            v-model="selectedSymptom"
                            class="mt-4 w-full rounded-md border border-white/10 bg-zinc-950 px-3 py-3 text-sm text-white outline-none transition focus:border-emerald-300 focus:ring-2 focus:ring-emerald-300/20"
                        >
                            <option v-for="fix in activeService.fixes" :key="fix.id" :value="fix.id">
                                {{ fix.label }}
                            </option>
                        </select>
                    </div>
                </aside>

                <section class="rounded-lg border border-white/10 bg-zinc-900/80 shadow-2xl shadow-black/40">
                    <div class="flex flex-col gap-3 border-b border-white/10 p-4 sm:flex-row sm:items-center sm:justify-between">
                        <div>
                            <div class="flex items-center gap-2 text-sm text-zinc-400">
                                <span class="size-3 rounded-full bg-rose-400"></span>
                                <span class="size-3 rounded-full bg-amber-300"></span>
                                <span class="size-3 rounded-full bg-emerald-400"></span>
                                <span class="ml-2 font-mono">devstrait://fix/{{ selectedOs }}/{{ activeService.id }}</span>
                            </div>
                            <h2 class="mt-3 text-xl font-semibold text-white">{{ activeFix.label }}</h2>
                        </div>
                        <button
                            type="button"
                            class="inline-flex items-center justify-center gap-2 rounded-md border px-4 py-2 text-sm font-semibold transition"
                            :class="copied ? 'border-emerald-300/50 bg-emerald-400/15 text-emerald-200' : 'border-white/10 bg-zinc-950 text-zinc-200 hover:border-emerald-300/50 hover:text-emerald-200'"
                            @click="copyCommands"
                        >
                            <CheckCircle2 v-if="copied" class="size-4" />
                            <Copy v-else class="size-4" />
                            {{ copied ? 'Copied' : 'Copy Code' }}
                        </button>
                    </div>

                    <div class="p-4">
                        <div class="mb-4 rounded-md border border-amber-300/20 bg-amber-300/10 p-4 text-sm leading-6 text-amber-100">
                            {{ activeFix.cause }}
                        </div>

                        <pre class="min-h-[360px] overflow-x-auto rounded-lg border border-white/10 bg-[#05070a] p-5 font-mono text-sm leading-7 text-emerald-200 shadow-inner"><code>{{ commandText }}</code></pre>

                        <div class="mt-4 grid gap-3 sm:grid-cols-3">
                            <div class="rounded-md border border-white/10 bg-zinc-950/70 p-3">
                                <p class="text-xs uppercase text-zinc-500">Operating System</p>
                                <p class="mt-1 text-sm font-semibold text-white">{{ activeOs.label }}</p>
                            </div>
                            <div class="rounded-md border border-white/10 bg-zinc-950/70 p-3">
                                <p class="text-xs uppercase text-zinc-500">Service</p>
                                <p class="mt-1 text-sm font-semibold text-white">{{ activeService.shortName }}</p>
                            </div>
                            <div class="rounded-md border border-white/10 bg-zinc-950/70 p-3">
                                <p class="text-xs uppercase text-zinc-500">Markdown Output</p>
                                <div class="mt-1 flex items-center gap-2 text-sm font-semibold text-emerald-300">
                                    <Clipboard class="size-4" />
                                    <span>{{ markdownOutput.split('\n').length }} lines ready</span>
                                </div>
                            </div>
                        </div>
                    </div>
                </section>
            </section>
        </div>
    </main>
</template>
