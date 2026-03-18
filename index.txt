// 🔹 套用模板函式
function renderTemplate(template, data) {
    if (!template) return "";
    return template.replace(/{{(\w+)}}/g, (_, key) => data[key] || "");
}

export default {
    async fetch(request, env, ctx) {
        const url = new URL(request.url);

        // 前端控制面板 API 路由
        if (url.pathname.startsWith("/api/")) {
            return handleDashboardAPI(request, env);
        }

        // Discord Bot 互動邏輯
        const signature = request.headers.get("x-signature-ed25519");
        const timestamp = request.headers.get("x-signature-timestamp");
        const body = await request.text();

        const isDev = env.IS_DEV === "true";

        let interaction;
        if (isDev) {
            interaction = {
                type: 2,
                data: { name: "luck" },
                member: { user: { id: "123", username: "測試用戶" } },
                channel_id: "1479520486856855686",
            };
        } else {
            const isValid = await verifyDiscordRequest(env.DISCORD_PUBLIC_KEY, signature, timestamp, body);
            if (!isValid) return new Response("Bad request signature", { status: 401 });

            try {
                interaction = JSON.parse(body);
            } catch {
                return new Response("Invalid JSON body", { status: 400 });
            }
        }

        console.log("收到指令:", interaction.data?.name || interaction.data?.custom_id);

        const username = interaction?.member?.nick
            || interaction?.member?.user?.global_name
            || interaction?.member?.user?.username
            || "你";
        const userId = interaction?.member?.user?.id || "unknown";
        const channelId = interaction?.channel_id || "unknown";

        // 從 KV 讀取前端設定
        const settings = JSON.parse(await env.SETTINGS.get("dashboard")) || {
            commands: { luck: true, lottery: true, wish: true, divination: true },
            luckOptions: ["大吉", "中吉", "吉", "小吉", "凶"],
            limits: {
                luck: { enabled: true, maxPerDay: 5 },
                lottery: { enabled: true, maxPerDay: 5 },
                wish: { enabled: true, maxPerDay: 3 },
                divination: { enabled: true, maxPerDay: 10 },
            },
            channels: {
                luck: "0",
                lottery: "0",
                wish: "0",
                divination: "0",
            }
        };

        // PING 驗證
        if (interaction.type === 1) {
            return jsonResponse({ type: 1 });
        }

        // Slash Command
        if (interaction.type === 2) {
            const command = interaction.data.name;
            // console.log(`使用者 ${username} 發出指令: ${command}`);
            // 檢查是否開放
            if (!settings.commands[command]) {
                return respondPublic(`${command} 功能目前已關閉`);
            }

            // 檢查頻道限制
            const channelMessages = {
                luck: "求籤要去土地公廟！",
                lottery: "抽號碼去抽號碼區！",
                wish: "許願池只能在指定頻道許願喔！",
                divination: "擲筊要在擲筊的地方才能進行！"
            };

            const allowedChannel = settings.channels?.[command] || "0";
            if (allowedChannel !== "0" && allowedChannel !== channelId) {
                const customMsg = channelMessages[command] || `${command} 功能只能在指定頻道使用`;
                return respondPublic(`${customMsg}`);
            }

            // 檢查限制
            const limitCheck = await checkLimit(env, command, userId, settings.limits[command]);
            if (!limitCheck.allowed) {
                return respondPublic(`${username} 今天的 ${command} 次數已達上限 (${settings.limits[command].maxPerDay} 次)`);
            }

            const usageInfo = `\n*（今日已使用 ${limitCheck.used}/${settings.limits[command].maxPerDay} 次）*`;

            if (command === "luck") {
                const fortunes = settings.luckOptions;
                const result = fortunes[Math.floor(Math.random() * fortunes.length)];

                return respondWithButton(
                    `${username}今天的運勢是：${result}${usageInfo}`,
                    "luck_confirm",
                    "再抽一次"
                );
            }


            if (command === "lottery") {
                const number = Math.floor(Math.random() * 100) + 1;
                const content = renderTemplate(settings.templates?.lotteryStart, {
                    username,
                    number,
                    used: limitCheck.used,
                    max: settings.limits.lottery.maxPerDay
                }) || `${username}抽到的號碼是：${number}\n*（今日已使用 ${limitCheck.used}/${settings.limits.lottery.maxPerDay} 次）*`;

                return respondWithButton(content, "lottery_confirm", "再抽一次");
            }

            if (command === "wish") {
                const wish = interaction.data.options?.[0]?.value || "未輸入願望";

                // 👉 呼叫 checkLimit 更新次數
                const limitCheck = await checkLimit(env, command, userId, settings.limits[command]);
                if (!limitCheck.allowed) {
                    return respondPublic(`${username} 今天的願望次數已達上限 (${settings.limits.wish.maxPerDay} 次)`);
                }

                const usageInfo = `（今日已使用 ${limitCheck.used}/${settings.limits.wish.maxPerDay} 次）`;

                return jsonResponse({
                    type: 4,
                    data: {
                        content: `**${username}的願望是：${wish}** ${usageInfo}`,
                        components: [
                            {
                                type: 1,
                                components: [
                                    { type: 2, style: 1, label: "送出願望", custom_id: "wish_confirm" },
                                ],
                            },
                        ],
                    },
                });
            }

            if (command === "divination") {
                const results = [
                    { name: "聖筊", image: "http://i.imgur.com/VbjWyGX.jpg" },
                    { name: "蓋筊", image: "http://i.imgur.com/mhLyrZE.jpg" },
                    { name: "笑筊", image: "http://i.imgur.com/o7MACE6.jpg" },
                ];
                const result = results[Math.floor(Math.random() * results.length)];
                const content = renderTemplate(settings.templates?.divinationStart, {
                    username,
                    result: result.name,
                    used: limitCheck.used,
                    max: settings.limits.divination.maxPerDay
                }) || `${username}擲筊結果是：${result.name}\n*（今日已使用 ${limitCheck.used}/${settings.limits.divination.maxPerDay} 次）*`;

                return respondWithEmbed(content, result.name, result.image, "divination_confirm", "再擲一次");
            }

        }

        // 🔹 按鈕互動
        if (interaction.type === 3) {
            const commandMap = {
                "luck_confirm": "luck",
                "lottery_confirm": "lottery",
                "wish_confirm": "wish",
                "divination_confirm": "divination"
            };

            const command = commandMap[interaction.data.custom_id];
            if (!command) {
                return new Response("Unknown button interaction", { status: 400 });
            }

            if (!settings.commands[command]) {
                return respondPublic(`${command} 功能目前已關閉`);
            }

            // 檢查頻道限制
            const allowedChannel = settings.channels?.[command] || "0";
            if (allowedChannel !== "0" && allowedChannel !== channelId) {
                return respondPublic(`${command} 功能只能在指定頻道使用 (ID: ${allowedChannel})`);
            }

            // 👉 呼叫一次 checkLimit
            const limitCheck = await checkLimit(env, command, userId, settings.limits[command]);
            if (!limitCheck.allowed) {
                return respondPublic(`${username} 今天的 ${command} 次數已達上限 (${settings.limits[command].maxPerDay} 次)`);
            }

            const usageInfo = `\n*（今日已使用 ${limitCheck.used}/${settings.limits[command].maxPerDay} 次）*`;

            // luck
            if (command === "luck") {
                const fortunes = settings.luckOptions;
                const result = fortunes[Math.floor(Math.random() * fortunes.length)];

                return respondWithButton(
                    `再抽一次！${username}今天的運勢是：${result}${usageInfo}`,
                    "luck_confirm",
                    "再抽一次"
                );
            }

            // lottery
            if (command === "lottery") {
                const number = Math.floor(Math.random() * 100) + 1;
                const content = renderTemplate(settings.templates?.lotteryConfirm, {
                    username,
                    number,
                    used: limitCheck.used,
                    max: settings.limits.lottery.maxPerDay
                }) || `再抽一次！${username}抽到的號碼是：${number}\n*（今日已使用 ${limitCheck.used}/${settings.limits.lottery.maxPerDay} 次）*`;

                return respondWithButton(content, "lottery_confirm", "再抽一次");
            }

            // wish
            if (command === "wish") {
                return respondPublic(settings.templates?.wishConfirm || "✨ 願望已送出！");
            }

            // divination
            if (command === "divination") {
                const results = [
                    { name: "聖筊", image: "http://i.imgur.com/VbjWyGX.jpg" },
                    { name: "蓋筊", image: "http://i.imgur.com/mhLyrZE.jpg" },
                    { name: "笑筊", image: "http://i.imgur.com/o7MACE6.jpg" },
                ];
                const result = results[Math.floor(Math.random() * results.length)];
                const content = renderTemplate(settings.templates?.divinationConfirm, {
                    username,
                    result: result.name,
                    used: limitCheck.used,
                    max: settings.limits.divination.maxPerDay
                }) || `再擲一次！${username}擲筊結果是：${result.name}\n*（今日已使用 ${limitCheck.used}/${settings.limits.divination.maxPerDay} 次）*`;

                return respondWithEmbed(content, result.name, result.image, "divination_confirm", "再擲一次");
            }
        }
        return new Response("Unhandled interaction", { status: 400 });
    },
};

// 🔹 Dashboard API 處理
async function handleDashboardAPI(request, env) {
    const url = new URL(request.url);

    // 處理 CORS 預檢請求
    if (request.method === "OPTIONS") {
        return new Response(null, {
            status: 204,
            headers: {
                "Access-Control-Allow-Origin": "*",
                "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
                "Access-Control-Allow-Headers": "Content-Type",
            },
        });
    }

    // 更新 KV (合併更新)
    if (url.pathname === "/api/updateKV" && request.method === "POST") {
        try {
            const newData = await request.json();
            const oldRaw = await env.SETTINGS.get("dashboard");
            const oldData = oldRaw ? JSON.parse(oldRaw) : {};

            // 合併舊資料與新資料
            const merged = {
                ...oldData,
                ...newData,
                commands: { ...oldData.commands, ...newData.commands },
                limits: { ...oldData.limits, ...newData.limits },
                channels: { ...oldData.channels, ...newData.channels },
            };

            await env.SETTINGS.put("dashboard", JSON.stringify(merged));

            return new Response(JSON.stringify({ success: true, data: merged }), {
                headers: {
                    "content-type": "application/json",
                    "Access-Control-Allow-Origin": "*",
                },
            });
        } catch (err) {
            return new Response(JSON.stringify({ success: false, error: err.message }), {
                headers: {
                    "content-type": "application/json",
                    "Access-Control-Allow-Origin": "*",
                },
                status: 500,
            });
        }
    }

    // 讀取 KV
    if (url.pathname === "/api/getKV" && request.method === "GET") {
        const raw = await env.SETTINGS.get("dashboard");
        const settings = raw ? JSON.parse(raw) : {};

        return new Response(JSON.stringify(settings), {
            headers: {
                "content-type": "application/json",
                "Access-Control-Allow-Origin": "*",
            },
        });
    }

    return new Response("Not Found", { status: 404 });
}

// 🔹 檢查限制 (每日次數)
async function checkLimit(env, command, userId, setting) {
    if (!setting || !setting.enabled) {
        return { allowed: true, remaining: Infinity, used: 0 };
    }

    const today = new Date();
    const todayKey = `${command}:${userId}:${today.getFullYear()}-${today.getMonth() + 1}-${today.getDate()}`; const used = parseInt(await env.SETTINGS.get(todayKey)) || 0;

    if (used >= setting.maxPerDay) {
        return { allowed: false, remaining: 0, used };
    }

    await env.SETTINGS.put(todayKey, (used + 1).toString(), { expirationTtl: 86400 });
    return { allowed: true, remaining: setting.maxPerDay - (used + 1), used: used + 1 };
}

// 🔹 回覆帶按鈕
function respondWithButton(content, customId, label) {
    return jsonResponse({
        type: 4,
        data: {
            content,
            components: [
                {
                    type: 1,
                    components: [
                        { type: 2, style: 1, label, custom_id: customId },
                    ],
                },
            ],
        },
    });
}

// 🔹 回覆帶圖片 embed + 按鈕
function respondWithEmbed(content, title, imageUrl, customId, label) {
    return jsonResponse({
        type: 4,
        data: {
            content,
            embeds: [{ title, image: { url: imageUrl } }],
            components: [
                {
                    type: 1,
                    components: [
                        { type: 2, style: 1, label, custom_id: customId },
                    ],
                },
            ],
        },
    });
}

// 🔹 公開訊息
function respondPublic(content) {
    return jsonResponse({ type: 4, data: { content } });
}

// 🔹 統一 JSON Response
function jsonResponse(obj) {
    return new Response(JSON.stringify(obj), {
        headers: { "Content-Type": "application/json" },
    });
}

// 🔹 驗證簽名
async function verifyDiscordRequest(publicKey, signature, timestamp, body) {
    if (!publicKey || !signature) return false;

    const encoder = new TextEncoder();
    const data = encoder.encode(timestamp + body);

    const keyData = hexToUint8Array(publicKey);
    const sigData = hexToUint8Array(signature);

    const cryptoKey = await crypto.subtle.importKey(
        "raw",
        keyData,
        { name: "Ed25519" },
        false,
        ["verify"]
    );

    return await crypto.subtle.verify("Ed25519", cryptoKey, sigData, data);
}

// 🔹 Hex 字串轉 Uint8Array
function hexToUint8Array(hex) {
    return new Uint8Array(hex.match(/.{1,2}/g).map((b) => parseInt(b, 16)));
}